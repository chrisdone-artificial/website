---
date: 2015-06-19
title: The constraint trick for instances
description: The constraint trick for instances
author: Chris Done
tags: haskell
---

Ever seen this in a library,

``` haskell
instance (var ~ AType) => ClassName (SomeType var)
```

and thought, “Shenanigans! Why not just have this?”

``` haskell
instance ClassName (SomeType AType)
```

Me too!

I only learned of this solution relatively recently, and I
know experienced Haskellers who also only understood this recently or
still don't. Hence this quick write up. Here's the thought process.

We're writing a trivial pretty printer and we're using `Writer`. We
write things like:

``` haskell
λ> execWriter (do tell "hello"; tell "world" :: Writer String ())
"helloworld"
```

Quality. But writing `tell` every time is so boring! How about we use
the `IsString` class so that we can just write the string literals
like this?

``` haskell
do "hello"; "world"
```

Let's write the `IsString` instance:

``` haskell
instance IsString (Writer String a) where
  fromString = tell
```

What do you say, GHC?

> Couldn't match type ‘a’ with ‘()’
>
> ‘a’ is a rigid type variable bound by the instance declaration

Oh. Good point. The type of our `tell` call results in `Writer String
()`. A small set back. Fine, let's change the instance declaration to
just be `()`:

``` haskell
instance IsString (Writer String ()) where
  fromString = tell
```

GHC loves it!

Let's try using it:

``` haskell
λ> execWriter (do "hello"; "world" :: Writer String ())
<interactive>:42:16:
    No instance for (IsString (WriterT String Identity a))
      arising from the literal ‘"hello"’
    The type variable ‘a’ is ambiguous
```

This displeases me. But it adds up given the type of `(>>)`:

``` haskell
(>>) :: Monad m => m a -> m b -> m b
```

In `_ >> return () :: Writer String ()`, the type of `_` is `Writer
String a`, so we really need an `IsString` instance that matches
that. But we *already* tried that. Oh, woe*!*

Some people reading this will be nodding in recognition of this same
problem they had while writing that perfect API that *just* won't work
because of this niggling issue.

Here comes the trick.[^1] So let's go back to a basic instance:

``` haskell
data MyTuple a b = MyTuple a b
instance Show (MyTuple a b) where
  show _ = "MyTuple <some value> <some value>"
```

Suppose I replace this instance with a new instance that has
constraints:

``` haskell
instance (Show a,Show b) => Show (MyTuple a b) where
  show (MyTuple a b) = "MyTuple " ++ show a ++ " " ++ show b
```

Question: Does that change whether GHC decides to pick this new
version of instance over others that may be available, compared to the
one above? Have a think.

The answer is: nein! The constraints of an instance don't have
anything to do with deciding whether an instance is picked from the
list of instances available. Constraints only apply *after GHC has
already decided it's going with this instance*.

So, cognizant of this obvious-after-the-fact property, let's use the
equality constraint that was introduced with GADTs and type families
(enabling either brings in `~`):

``` haskell
instance a ~ () => IsString (Writer String a) where
  fromString = tell
```

Let's try it:

``` haskell
λ> execWriter (do "hello" ; "world" :: Writer String ())
"helloworld"
```

This instance is picked by GHC, as we hoped, because of the `a`. The
instance method also type checks, because the constraint applies when type
checking the instance methods, just like if you write a regular
declaration like:

``` haskell
foo :: (a ~ ()) => a
foo = ()
```

That's it! This crops up in a number of my own libraries and knowing
this really helped me. Here is
[a real example from my `lucid` library](https://github.com/chrisdone/lucid/blob/16c4b09ee90e2e61fef0e1070c0c3ebaf6246da1/src/Lucid/Base.hs#L81):

``` haskell
instance (a ~ (),Monad m) => Monoid (HtmlT m a) where
  mempty  = return mempty
  mappend = liftM2 mappend
```

Hope this was helpful!

[^1]: Actually, it's a natural consequence to grokking how instance
      resolution works (but calling it a "trick" makes for a catchy
      title).
