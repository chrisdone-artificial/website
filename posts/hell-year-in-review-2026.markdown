---
date: 2026-04-23
title: "Some updates about Hell in 2026"
---

I've added a few neat bits and bobs to Hell since I last blogged about
it.

On the API side, things have slowed down when compared with the prior
year,[^1] but other improvements were still made.

## Sum types

I've added support for using case statements on built-in primitive
types, like `Maybe`, as in:

```haskell
case i of
  Maybe.Just x -> IO.print x
  Maybe.Nothing -> Text.putStrLn "nope"
```

Previously, one only had `Maybe.maybe`, which is fine, but is often
not the direct way to write something. So having `case` syntax for
that is handy. You just have to qualify the constructor names, and
then it knows that it's a built-in.

## Type class entailment

I’ve added limited support for resolving instances which feature
entailment on the instance, such as `Eq a => Eq [a]`, and this works in
a nested way. So one can compare values of any depth provided they
have an `Eq` instance, as seen in the following examples:

```haskell
Text.putStrLn $ Show.show $ Eq.eq (1,1) (1,1)
Text.putStrLn $ Show.show $ Eq.eq [Maybe.Just 1] [Maybe.Just 2]
Text.putStrLn $ Show.show $ Eq.eq [Either.Left 1] [Either.Right "abc"]
IO.print [Maybe.Just 1, Maybe.Nothing]
IO.print $ Maybe.Just [1] <> Maybe.Nothing
IO.print $ [Either.Left (Maybe.Just 1), Either.Right (Maybe.Just "abc"), Either.Left Maybe.Nothing]
IO.print [Maybe.Just (1, 2), Maybe.Nothing]
```

The complete list of instances is now on [the API documentation
page](https://chrisdone.github.io/hell/api/).

The implementation is quite nice, where instances are listed like
this:

```haskell
instances :: Instances
instances =
  Instances $
    Map.fromList
      [ entail1 @Show @[],
        entail1 @Show @Set,
        entail1 @Show @Tree,
        entail1 @Show @Maybe,
        entail1 @Show @Vector,
        entail2 @Show @Either,
        entail2 @Show @(,),
        ...
        instance1 @Functor @Either,
        instance1 @Functor @(,), -- Functor (a,)
        instance0 @Applicative @IO,
        instance0 @Applicative @Maybe,
        instance0 @Applicative @[],
        instance0 @Applicative @Tree,
        instance0 @Applicative @Options.Parser,
        instance1 @Applicative @Either,
        instance0 @Alternative @Options.Parser,
        instance0 @Alternative @Maybe,
        entail1 @Monoid @Maybe,
        instance0 @Monoid @Text,
        instance1 @Monoid @Vector,
        instance2 @Monoid @Options.Mod,
        instance1 @Monoid @[],
        entail1 @Semigroup @Maybe,
        instance2 @Semigroup @Either,
        instance2 @Semigroup @Options.Mod,
        instance0 @Semigroup @Text,
        ...
```

With

```haskell
instance0 ::
  forall cls a.
  (cls a, Typeable cls, Typeable a) =>
  ((SomeTypeRep, SomeTypeRep), Dynamic)
instance0 =
  ( (SomeTypeRep $ typeRep @cls, SomeTypeRep $ typeRep @a),
    toDyn $ Dict @(cls a)
  )

instance1 ::
  forall {k0} {k1} (c :: k1 -> Constraint) (t :: k0 -> k1).
  ((forall a. c (t a)), Typeable c, Typeable t, Typeable k0, Typeable k1) =>
  ((SomeTypeRep, SomeTypeRep), Dynamic)
instance1 =
  ( (SomeTypeRep $ typeRep @c, SomeTypeRep $ typeRep @t),
    toDyn $ D1 @c @t Dict
  )

entail1 ::
  forall {k1} (c :: k1 -> Constraint) (t :: k1 -> k1).
  ((forall a. c a => c (t a)), Typeable c, Typeable t,  Typeable k1) =>
  ((SomeTypeRep, SomeTypeRep), Dynamic)
entail1 =
  ( (SomeTypeRep $ typeRep @c, SomeTypeRep $ typeRep @t),
    toDyn $ ED1 @c @t (Sub Dict)
  )

-- Entailment, c a => c (t a), E.g. Eq a :- Eq [a]
newtype ED1 c t = ED1 (forall e. c e :- c (t e))
```

And then resolution looks like this:

```haskell
resolve1 ::
  forall {k0} {k1} (t :: k0 -> k1) (c :: k1 -> Constraint) (a :: k0).
  (Typeable k0, Typeable k1) =>
  TypeRep (c (t a)) ->
  TypeRep c ->
  TypeRep t ->
  Instances ->
  Maybe (Dict (c (t a)))
resolve1 cta c t (Instances m) = do
  Dynamic rep dict <- Map.lookup (SomeTypeRep c, SomeTypeRep t) m
  (do Type.HRefl <- Type.eqTypeRep rep $ Type.App (Type.App (typeRep @D1) c) t
      let D1 d = dict
      pure d) <|>
    -- When we see e.g. C (T A), where T A and A have the same kind,
    -- we can lookup C A, for the entailment C A :- C (T A).
    (do case cta of
          Type.App _c a@(Type.App f a') -> do
            Type.HRefl <- Type.eqTypeRep (typeRepKind a') (typeRepKind a)
            Type.HRefl <- Type.eqTypeRep rep $ Type.App (Type.App (typeRep @ED1) c) f
            let ED1 entailment = dict
            dictA <- lookupDict a' c
            pure $ mapDict entailment dictA
          _ -> Nothing)
```

For future reference, you can see the latest implementation [in Hell.hs in the repo.](https://github.com/chrisdone/hell/blob/main/src/Hell.hs)

What remains is adding support for e.g. `Ord a => Eq (Set a)`, but I
don't expect it'll be too hard.

## APIs added

I added optparse-applicative, so the following code works exactly as
it would in Haskell:

```haskell
-- Includes example of Semigroup.
data Opts = Opts {
  quiet :: Bool,
  filePath :: Maybe Text
 }
options =
  (\quiet path -> Main.Opts { quiet = quiet, filePath = path })
       <$> Options.switch (Flag.long "quiet" <> Flag.help "Be quiet?")
       <*> (Alternative.optional $ Options.strOption (Option.long "path" <> Option.help "The filepath to export"))
main = do
  opts <- Options.execParser (Options.info (Main.options <**> Options.helper) Options.fullDesc)
  Text.putStrLn $ Maybe.maybe "No file path" Function.id (Record.get @"filePath" opts)
  Text.putStrLn $ Show.show @Bool $ Record.get @"quiet" opts
```

It's got date/time APIs:

```haskell
main = do
  now <- UTCTime.getCurrentTime
  Text.putStrLn "Current time:"
  IO.print now

  Text.putStrLn "ISO8601:"
  Text.putStrLn $ UTCTime.iso8601Show now

  Text.putStrLn "Parsed:"
  Maybe.maybe (Error.error "Impossible!") IO.print $ UTCTime.iso8601ParseM "2025-05-30T11:18:26.195147084Z"

  Text.putStrLn "Increased:"
  IO.print $ UTCTime.addUTCTime (Double.mult 60.0 60.0) now

  Text.putStrLn "Parts:"
  IO.print $ TimeOfDay.timeToTimeOfDay $ UTCTime.utctDayTime now
  IO.print $ UTCTime.utctDay now

  day1 :: Day <-
    Maybe.maybe (Error.error "Invalid") IO.pure $ Day.fromGregorianValid (Int.toInteger 2025) 08 09
  day2 <- Maybe.maybe (Error.error "Invalid") IO.pure $ Day.iso8601ParseM "2025-08-09"
  IO.print $ Eq.eq day1 day2 -- True
  Text.putStrLn $ Day.iso8601Show day1 -- 2025-08-09
```

I think Haskell has a really nice date/time API -- compared to most
other languages, anyway.

I've been planning for my next thing to add to be Wai/Warp, so that
it's easy to write HTTP servers which comes up surprisingly often. The
Wai/Warp API is very stable, so I can feel happy including it as a
primitive.

## Syntactic constructs added

I added the ability to declare type sigs on top-levels like this:

```haskell
data Foo = Foo { bar, mu :: Int }
main :: IO () =
  Main.foo
```

Also added applicative operators, and NamedFieldPuns. I think that's
about it.

## Remaining work

I need to implement a faster unification—I have a work in progress
unification-fd branch, which I may or may not proceed with, as the API
doesn't bring me much joy. Some speed up is needed, though, as my
naive algorithm is slow.

It still needs to be ported to the GHC parser, so we can get all the
extra syntactic goodies like foo.bar, multi-line strings, etc. But
that's boring, labrious work, so I'll get to it when I really, really
want to.

No LLMs have been used on any part of this project so far. It's more
of an art project at this point. My artisanal open source project.

[^1]: For which a lot of activity was driven by adding more library
functions--because our team at work has been adopting Nushell
alongside Hell.  Nushell is in many ways is easier than Hell, for
things like data munging and web requests and such out of the box, and
it has a proper REPL. Hell doesn't have a REPL and isn't trying to be
that kind of experience.

It was debated whether we should just use Nushell for everything, but
a couple people argued that Hell is particularly good at process
orchestration due to its simple and easy concurrency primitives. So
we're using both for now.
