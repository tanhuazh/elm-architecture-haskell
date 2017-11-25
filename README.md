# elm-architecture-haskell

This is an implementation of the Elm Architecture in Haskell.

This implementation runs commands concurrently just like in Elm.

Internally commands run in separate threads. If any of the threads errors, the
error will be re-thrown in the main thread.

## Implementation

```haskell
{-# LANGUAGE NoImplicitPrelude   #-}
{-# LANGUAGE RankNTypes          #-}
{-# LANGUAGE ScopedTypeVariables #-}

--------------------------------------------------------------------------------
import           Control.Concurrent       (threadDelay)
import           Control.Concurrent.Async (Async, async, waitAny)
--------------------------------------------------------------------------------

type Cmd a = [ IO a ]


data Config model msg =
    Config
    { _init   :: (model, Cmd msg)
    , _update :: msg -> model -> (model, Cmd msg)
    }


runProgram :: forall model msg. Config model msg -> IO ()
runProgram config =
    do
        initAsyncs <- traverse async initCmds
        run' initAsyncs initModel
    where
        update' = _update config

        (initModel, initCmds) = _init config

        run' :: [Async msg] -> model -> IO ()
        run' asyncs model =
            if null asyncs then
                print "Finished"
            else
                do
                    -- This works like a pool of async commands with a queue
                    -- in the end. The first command to be resolved is the first
                    -- command dealt with.
                    (completedCmd, msg) <- waitAny asyncs :: IO (Async msg, msg)

                    let (newModel, newCmds) = update' msg model

                    newCmdsAsync <- traverse async newCmds

                    let newAsyncs =
                            filter (/= completedCmd) -- Remove the cmd that we just ran
                            asyncs ++ newCmdsAsync   -- Let's add what our update returned

                    run' newAsyncs newModel
```

## Example Program

```haskell
-- Program output :
--     Program started
--     New model value: Model 0
--     New model value: Model 2
--     New model value: Model 3

-- ===================
-- MAIN
-- ===================

main :: IO ()
main =
    runProgram Config
        { _init = init
        , _update = update
        }


-- ===================
-- TYPES
-- ===================


newtype Model = Model Int
    deriving (Show)


data Msg
    = DoNothing
    | SetModelValue Int
    | WaitAndSetModel Int Int

-- ===================
-- UPDATE
-- ===================


init :: (Model, Cmd Msg)
init =
    (Model 0
    ,   [ return $ WaitAndSetModel 2 2 -- waits 2 seconds and sets model value to 2
        , return $ WaitAndSetModel 5 5
        , return $ WaitAndSetModel 3 3
        , cmdIO $ putStrLn "Program started"
        ]
    )


update :: Msg -> Model -> (Model, Cmd Msg)
update msg model =
    case msg of
        DoNothing ->
            (model, [])

        SetModelValue newVal ->
            let
                newModel = Model newVal
                msgToPrint = "New model value: " ++ show model
            in
                (newModel, return $ cmdIO $ putStrLn msgToPrint)

        WaitAndSetModel secs val ->
            let
                cmd = do
                    threadDelay $ secs * 1000000
                    return $ SetModelValue val
            in
                (model, [cmd])


cmdIO :: IO a -> IO Msg
cmdIO io = io >> return DoNothing
```
