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
Then it’s easy to graph it out or batch it as needed. Works for SQL DBs (e.g. Rel8), 
build systems or FRP (e.g. Reflex).

This does mean you can’t simply run `mapM` against a `Value [a]`, 
and this often requires a special operator for the action in the domain in question.

I’m not sure that there’s already name for it, but it’s definitely a pattern. 
You see it in quite a few places. Hence pointing it out.

Full code example below. I've added an extra `f` parameter on the `Action` type to emphasize that
by exposing an API like this, you can inspect the monad's structure, but that you can also
keep your monad well-formed (i.e. interpret the `Applicative` as `Identity`). But it's not necessary
to add a parameter to everything, and in practice all examples I've seen do not have a parameter, 
and are specialized on a particular type. Sometimes the type isn't actually an `Applicative`, 
but is similar in spirit (e.g. DB libraries often have `Expr a` returned by `Query a` monad).

```haskell
{-# language KindSignatures, BlockArguments, GADTs, LambdaCase, GeneralizedNewtypeDeriving #-}
import Data.Functor.Coyoneda
import Data.Kind
import Data.Functor.Const
import Control.Monad.Free
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
  Spec :: String -> f (m a) -> Spec f m (f a)

newtype Action f m a = Action { runAction :: Free (Coyoneda (Spec f m)) a }
  deriving (Functor, Applicative, Monad)

act :: String -> f (m a) -> Action f m (f a)
act l f = Action $ liftF $ liftCoyoneda $ Spec l f

--------------------------------------------------------------------------------
-- An example

example :: Applicative f => Action f IO (f (ByteString, ByteString))
example = do
  file1 <- act "read_file_1" (pure $ S.readFile "file1.txt")
  file2 <- act "read_file_2" (S.readFile .  unwords . words . S8.unpack <$> file1)
  pure $ (,) <$> file1 <*> file2

--------------------------------------------------------------------------------
-- IO interpretation

runIO :: Action Identity IO a -> IO a
runIO = foldFree (lowerCoyoneda . hoistCoyoneda io) . runAction where
  io :: Spec Identity IO x -> IO x
  io = \case
    Spec name act' -> do
      putStrLn $ "Running " ++ name
      out <- runIdentity act'
      pure $ Identity out

--------------------------------------------------------------------------------
-- Graphable interpretation

newtype Value (a :: Type) = Value { runValue :: Const (Set String) a }
  deriving (Functor, Applicative)
instance Show (Value a) where
  show = show . keys

runGraph :: Applicative m => Action Value m (Value a) -> (Value a, Map String (Set String))
runGraph x = flip runState mempty $ graph $ do
  v <- x
  act "root" (pure <$> v)

graph :: Action Value m a -> State (Map String (Set String)) a
graph = foldFree (lowerCoyoneda . hoistCoyoneda go) . runAction where
  go :: Spec Value m a -> State (Map String (Set String)) a
  go = \case
   Spec string i -> do
    modify (Map.insert string (keys i))
    pure $ Value $ Const (Set.singleton string)

keys ::  Value a -> Set String
keys = getConst . runValue
```

Example:

```haskell
-- Example:

-- Run as raw IO:

-- > runIO example
-- Running read_file_1
-- Running read_file_2
-- Identity ("hello.txt\n","you found me\n")

-- Dependency graph:

-- > runGraph example
-- (fromList ["root"],fromList [("read_file_1",fromList []),("read_file_2",fromList ["read_file_1"]),("root",fromList ["read_file_1","read_file_2"])])
```

Thanks to Liam Goodacre for his suggestion to eliminate the earlier inlined Coyoneda in my example.
Thanks to Jack Kelly for his suggestion to use a `Coyoneda` rather than free-Applicative, as this example doesn't need it.
