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
