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
{-# language GADTs, LambdaCase #-}
import qualified Data.ByteString as S
import qualified Data.ByteString.Char8 as S8
import Data.Functor.Identity
import Data.ByteString (ByteString)
import qualified Data.Map as Map
import Data.Map (Map)
import qualified Data.Set as Set
import Data.Set (Set)
import Control.Monad.Trans.State.Strict
import Control.Monad

--------------------------------------------------------------------------------
-- The applicative-wired monad pattern

data Action f m a where
  Return :: a -> Action f m a
  Bind :: Action f m a -> (a -> Action f m b) -> Action f m b
  Action :: String -> f i -> (i -> m a) -> Action f m (f a)

instance Monad (Action f m) where return = pure; (>>=) = Bind
instance Applicative (Action f m) where (<*>) = ap; pure = Return
instance Functor (Action f m) where fmap = liftM

--------------------------------------------------------------------------------
-- An example

example :: Applicative f => Action f IO (f (ByteString, ByteString))
example = do
  file1 <- Action "read_file_1" (pure ()) $ const $ S.readFile "file1.txt"
  file2 <- Action "read_file_2" file1 $ S.readFile .  unwords . words . S8.unpack
  pure $ (,) <$> file1 <*> file2

--------------------------------------------------------------------------------
-- IO interpretation

runIO :: Action Identity IO a -> IO a
runIO = \case
  Return a -> return a
  Bind m f -> runIO m >>= runIO . f
  Action name input act -> do
    putStrLn $ "Running " ++ name
    out <- act (runIdentity input)
    pure $ Identity out

--------------------------------------------------------------------------------
-- Graphable interpretation

data Value a where
  Key :: String -> Value a
  Pure :: a -> Value a
  Ap :: Value (a -> b) -> Value a -> Value b

instance Applicative Value where (<*>) = Ap; pure = Pure
instance Functor Value where fmap f m = pure f <*> m

graph :: Action Value m a -> State (Map String (Set String)) a
graph = \case
  Action string i _ -> do
    modify (Map.insert string (keys i))
    pure $ Key string
  Bind m f -> graph m >>= graph . f
  Return a -> pure a

keys :: Value a -> Set String
keys = \case
  Pure _ -> mempty
  Key k -> Set.singleton k
  Ap f m -> keys f <> keys m
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
