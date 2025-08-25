---
title: Applicative-wired monad pattern
date: 2025-07-11
description: Different ways of writing code
---

In Haskell API design, you sometimes want to model a computation that looks like a monad, 
i.e. some things depend on other things, and make use of do-notation, 
but you want to be able to statically inspect the resulting structure, too. 

The `ApplicativeDo` notation attempts to bridge this gap by a language extension and some conventions. 
It lets you write applicatives with do-notation, **but** dependencies between actions are **explicitly forbidden.** 
That limits its utility for this purpose.

Here’s a separate pattern I’ve put to use in a build system library, 
but has been used in popular database libraries and FRP libraries, 
before I ever did.

You have two types like,

```haskell
data Action a
data Value a
```

The `Action` is some instance of `Monad`, and could be a free monad. 
The `Value` is some instance of `Applicative`, and can be a free applicative.

The trick is that all functions exposed by the API only return the type `Action (Value a)`, 
and sometimes accept `Value a` as arguments. This means you wire up a graph, 
with `Action` containing nodes and `Value` serving the edges. 
You combine multiple output values into a single argument value via its `Applicative` instance. 
Then it’s easy to either run it as a regular action (by interpreting `Value` as `Identity`), 
or graph it out or batch it as needed.  Works for SQL DBs (e.g. Rel8), build systems or FRP (e.g. Reflex).

This does mean you can’t simply run `mapM` against a `Value [a]`, 
and this often requires a special operator for the action in the domain in question.

I’m not sure that there’s already name for it, but it’s definitely a pattern. 
You see it in quite a few places. Hence pointing it out.

Full code example:

```haskell
{-# LANGUAGE KindSignatures #-}
{-# language GADTs, LambdaCase, GeneralizedNewtypeDeriving #-}
import Control.Monad.Free
import Control.Applicative.Free
import qualified Data.ByteString as S
import qualified Data.ByteString.Char8 as S8
import Data.Functor.Identity
import Data.ByteString (ByteString)
import qualified Data.Map as Map
import Data.Map (Map)
import qualified Data.Set as Set
import Data.Set (Set)
import Control.Monad.Trans.State.Strict

--------------------------------------------------------------------------------
-- The applicative-wired monad pattern

data Spec f m a where
  Spec :: String -> f i -> (i -> m a) -> Spec f m (f a)

newtype Action f m a = Action { runAction :: Free (Ap (Spec f m)) a }
  deriving (Functor, Applicative, Monad)

act :: String -> f i -> (i -> m a) -> Action f m (f a)
act l i f = Action $ liftF $ liftAp $ Spec l i f

--------------------------------------------------------------------------------
-- An example

example :: Applicative f => Action f IO (f (ByteString, ByteString))
example = do
  file1 <- act "read_file_1" (pure ()) $ const $ S.readFile "file1.txt"
  file2 <- act "read_file_2" file1 $ S.readFile .  unwords . words . S8.unpack
  pure $ (,) <$> file1 <*> file2

--------------------------------------------------------------------------------
-- IO interpretation

runIO :: Action Identity IO a -> IO a
runIO = foldFree (runAp io) . runAction where
  io :: Spec Identity IO x -> IO x
  io = \case
    Spec name input act' -> do
      putStrLn $ "Running " ++ name
      out <- act' $ runIdentity input
      pure $ Identity out

--------------------------------------------------------------------------------
-- Graphable interpretation

newtype Value a = Value { runValue :: Ap Key a }
  deriving (Functor, Applicative)

data Key a = Key { unKey :: String }

graph :: Monad m => Action Value m a -> State (Map String (Set String)) a
graph = foldFree (runAp go) . runAction where
  go :: Spec Value m a -> State (Map String (Set String)) a
  go = \case
   Spec string i _ -> do
    modify (Map.insert string (keys i))
    pure $ Value $ liftAp $ Key string

  keys ::  Value a -> Set String
  keys = runAp_ (Set.singleton . unKey) . runValue
```

Example:

```haskell
-- Run as raw IO:

> runIO example
Running read_file_1
Running read_file_2
Identity ("file2.txt\n","Second file!\n")

-- Dependency graph:

> flip execState mempty $ graph example
fromList [("read_file_1",fromList []),("read_file_2",fromList ["read_file_1"])]
```
