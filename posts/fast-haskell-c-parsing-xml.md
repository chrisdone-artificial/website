---
date: 2017-01-11
title: "Fast Haskell: Competing with C at parsing XML"
description: "Fast Haskell: Competing with C at parsing XML"
author: Chris Done
tags: haskell
---

In this post we're going to look at parsing XML in Haskell, how it
compares with an efficient C parser, and steps you can take in Haskell
to build a fast library from the ground up. We're going to get fairly
detailed and get our hands dirty.

<!-- establish context -->

## A new kid on the block

A few weeks ago Neil Mitchell posted
[a blog post](http://neilmitchell.blogspot.co.uk/2016/12/new-xml-parser-hexml.html)
about a new XML library that he'd written. The parser is written in C,
and the API is written in Haskell which uses the C library. He writes
that it's very fast:

> Hexml has been designed for speed. In the very limited benchmarks
> I've done it is typically just over 2x faster at parsing than
> Pugixml, where Pugixml is the gold standard for fast XML DOM
> parsers. In my uses it has turned XML parsing from a bottleneck to
> an irrelevance, so it works for me.

In order to achieve that speed, he cheats by not performing operations
he doesn't care about:

> To gain that speed, Hexml cheats. Primarily it doesn't do entity
> expansion, so `&amp;` remains as `&amp;` in the output. It also
> doesn't handle `CData` sections (but that's because I'm lazy) and
> comment locations are not remembered. It also doesn't deal with most
> of the XML standard, ignoring the `DOCTYPE` stuff. [..] I only work
> on UTF8, which for the bits of UTF8 I care about, is the same as
> ASCII - I don't need to do any character decoding.

Cheating is fine when you describe in detail how you cheat. That's
just changing the rules of the game!

<!-- why this is of interest to me -->

## But C has problems

This post caught my attention because it seemed to me a pity to use
C. Whether you use Haskell, Python, or whatever, there are a few
problems with dropping down to C from your high-level language:

* The program is more likely to segfault. I'll take an exception over
  a segfault any day!
* The program opens itself up to possible exploitation due to lack of
  memory safety.
* If people want to extend your software, they have to use C, and not
  your high-level language.
* Portability (i.e. Windows) is a pain in the arse with C.

Sure enough, it wasn't long before
[Austin Seipp posted a rundown of bugs in the C code](https://www.reddit.com/r/haskell/comments/5i2mg1/new_xml_parser_hexml/db5os2h/):

> At the moment, sorry to say -- I wouldn't use this library to parse
> any arbitrary XML, since it could be considered hostile, and get me
> owned.  Using American Fuzzy Lop, just after a few minutes, I've
> already found around ~30 unique crashes.

But C is really fast right? Like 100s of times faster than Haskell!
It's worth the risk.

## But-but C is fast!

Let's benchmark it. We're going to parse a 4KB, a 31KB and a 211KB XML
file.

Using the
[Criterion benchmarking package](http://www.serpentine.com/criterion/),
we can compare Hexml against the pretty old Haskell `xml` package...

[image deleted due to region politics]

    File   hexml    xml
	4KB    6.26 μs  1.94 ms (1940 μs)
    31KB   9.41 μs  13.6 ms (13600 μs)
    211KB  260  μs  25.9 ms (25900 μs)

Ouch! Those numbers don't look good. The xml package is 100-300x times
slower.

Okay, I'm being unfair. The xml package isn't known for speed. Its
package description is simply `A simple XML library.` Let's compare
with the hexpat package. That one has this in its description:

> The design goals are speed, speed, speed, interface simplicity and
> modularity.

So that's probably more representing the best in Haskell XML
parsers. It's also based on the C expat library, which is supposed to
be fast.

[image deleted due to region politics]

    File   hexml     hexpat
    4KB    6.395 μs  320.3 μs
    31KB   9.474 μs  378.3 μs
    211KB  256.2 μs  25.68 ms

That's a bit better. We're now between 40-100x slower than Hexml. I'd
prefer 10x slower, but it's a more reasonable outcome. The hexpat
package handles: keeping location information, reasonable parse
errors, the complete XML standard. Hexml doesn't do any of that.

Let's set us a challenge. Can we match or beat the Hexml package in
plain old Haskell? This is an itch that got under my skin. I emailed
Neil and he was fine with it:

> I don't think it's unfair or attacky to use Hexml as the baseline -
> I'd welcome it!

I'll walk you through my approach. I called my library
[Xeno](https://github.com/chrisdone/xeno) (for obvious reasons).

## Start with the simplest thing possible

...and make sure it's fast. Here's the first thing I wrote, to see how
fast it was to walk across a file compared with Hexml.


``` haskell
module Xeno (parse) where
import           Data.ByteString (ByteString)
import qualified Data.ByteString as S
import           Data.Word

-- | Parse an XML document.
parse :: ByteString -> ()
parse str =
  parseTags 0
  where
    parseTags index =
      case elemIndexFrom 60 str index of
        Nothing ->
          ()
        Just fromLt ->
          case elemIndexFrom 62 str fromLt of
            Nothing -> ()
            Just fromGt -> do
              parseTags (fromGt + 1)

-- | Get index of an element starting from offset.
elemIndexFrom :: Word8 -> ByteString -> Int -> Maybe Int
elemIndexFrom c str offset = fmap (+ offset) (S.elemIndex c (S.drop offset str))
{-# INLINE elemIndexFrom #-}
```

The numbers 60 and 62 are `<` and `>`. In XML the only characters that
matter are `<` and `>` (if you don't care about entities). `<` and `>`
can't appear inside speech marks (attributes). They are the only
important things to search for. Results:

    File   hexml     xeno
    4KB    6.395 μs  2.630 μs
    42KB   37.55 μs  7.814 μs

So the baseline performance of walking across the file in jumps is
quite fast! Why is it fast? Let's look at that for a minute:

* The `ByteString` data type is a safe wrapper around a vector of
  bytes. It's underneath equivalent to `char*` in C.
* With that in mind,
  [the `S.elemIndex` function](http://hackage.haskell.org/package/bytestring-0.10.8.1/docs/Data-ByteString.html#v:elemIndex)
  is implemented using the standard C function `memchr(3)`. As we all
  know, `memchr` jumps across your file in large word boundaries or
  even using SIMD operations, meaning it's bloody fast. But the
  `elemIndex` function itself is safe.

So we're effectively doing a `for(..) { s=memchr(s,..) }` loop over
the file.

## Keep an eye on the allocations

Using the [weigh package](https://github.com/fpco/weigh#readme) for
memory allocation tracking, we can also look at allocations of our code right now:

    Case         Bytes  GCs  Check
    4kb parse    1,168    0  OK
    42kb parse   1,560    0  OK
    52kb parse   1,168    0  OK
    182kb parse  1,168    0  OK

We see that it's constant. Okay, it varies by a few bytes, but it
doesn't increase linearly or anything. That's good! One thing that
stood out to me, is that didn't we pay for allocation of the `Maybe`
values. For a 1000x `<` and `>` characters, we should have 1000
allocations of `Just`/`Nothing`. Let's go down that rabbit hole for a
second.

## Looking at the Core

Well, if you compile the source like
this

    stack ghc -- -O2 -ddump-simpl Xeno.hs

You'll see a dump of the real
[Core](http://stackoverflow.com/questions/6121146/reading-ghc-core)
code that is generated after the Haskell code is desugared, and before
it's compiled to machine code. At this stage you can already see
optimizations based on inlining, common-sub-expression elimination,
deforestation, and other things.

The output is rather large. Core is verbose, and fast code tends to be
longer. [Here is the output](https://gist.github.com/chrisdone/7aea4748c6fd0f1854d341b5676a4a97),
but you don't have to understand it. Just note that there's no mention
of Maybe, Just or Nothing in there. It skips that altogether. See
[here specifically](https://gist.github.com/chrisdone/7aea4748c6fd0f1854d341b5676a4a97#file-1core-hs-L38-L48). There
is a call to `memchr`, then there is an `eqAddr` comparison with
`NULL`, to see whether the `memchr` is done or not. But we're
[still doing safety checks](https://gist.github.com/chrisdone/7aea4748c6fd0f1854d341b5676a4a97#file-1core-hs-L146)
so that the resulting code is safe.

## Inlining counts

The curious reader might have noticed that `INLINE` line in my first
code sample.

``` haskell
{-# INLINE elemIndexFrom #-}
```

Without the INLINE, the whole function is twice as slow and has linear
allocation.

    Case        Bytes  GCs  Check
    4kb parse   1,472    0  OK
    42kb parse  1,160    0  OK
    52kb parse  1,160    0  OK

    benchmarking 4KB/xeno
    time                 2.512 μs   (2.477 μs .. 2.545 μs)
    benchmarking 211KB/xeno
    time                 129.9 μs   (128.7 μs .. 131.2 μs)
    benchmarking 31KB/xeno
    time                 1.930 μs   (1.909 μs .. 1.958 μs)

versus:

    Case         Bytes  GCs  Check
    4kb parse   12,416    0  OK
    42kb parse  30,080    0  OK
    52kb parse  46,208    0  OK

    benchmarking 4KB/xeno
    time                 5.258 μs   (5.249 μs .. 5.266 μs)
    benchmarking 211KB/xeno
    time                 265.9 μs   (262.4 μs .. 271.4 μs)
    benchmarking 31KB/xeno
    time                 3.212 μs   (3.209 μs .. 3.218 μs)

Always pay attention to things like this. You don't want to put INLINE
on everything. Sometimes it adds slowdown, most times it makes no
difference. So check with your benchmark suite.

## Loop unrolling manually

Some things need to be done manually. I added comment parsing to our
little function:

``` haskell
+        Just fromLt -> checkOpenComment (fromLt + 1)
+    checkOpenComment index =
+      if S.isPrefixOf "!--" (S.drop index str)
+         then findCommentEnd (index + 3)
+         else findLt index
+    findCommentEnd index =
+      case elemIndexFrom commentChar str index of
+        Nothing -> () -- error!
+        Just fromDash ->
+          if S.isPrefixOf "->" (S.drop (fromDash + 1) str)
+             then findGt (fromDash + 2)
+             else findCommentEnd (fromDash + 1)
```

And it became 2x slower:

    benchmarking 4KB/xeno
    time                 2.512 μs   (2.477 μs .. 2.545 μs)

to

    benchmarking 4KB/xeno
    time                 4.296 μs   (4.240 μs .. 4.348 μs)

So I changed the `S.isPrefixOf` to be unrolled to `S.index` calls, like
this:

``` haskell
-      if S.isPrefixOf "!--" (S.drop index str)
-         then findCommentEnd (index + 3)
-         else findLt index
+      if S.index this 0 == bangChar &&
+         S.index this 1 == commentChar &&
+         S.index this 2 == commentChar
+        then findCommentEnd (index + 3)
+        else findLt index
+      where
+        this = S.drop index str
```

And it dropped back down to our base speed again.

## Finding tag names

I implemented finding tag names like this:

``` haskell
+    findTagName index0 =
+      case S.findIndex (not . isTagName) (S.drop index str) of
+        Nothing -> error "Couldn't find end of tag name."
+        Just ((+ index) -> spaceOrCloseTag) ->
+          if S.head this == closeTagChar
+            then findGt spaceOrCloseTag
+            else if S.head this == spaceChar
+                   then findLt spaceOrCloseTag
+                   else error
+                          ("Expecting space or closing '>' after tag name, but got: " ++
+                           show this)
+          where this = S.drop spaceOrCloseTag str
+      where
+        index =
+          if S.head (S.drop index0 str) == questionChar ||
+             S.head (S.drop index0 str) == slashChar
+            then index0 + 1
+            else index0
```

And immediately noticed a big slow down. From

    Case        Bytes  GCs  Check
    4kb parse   1,160    0  OK
    42kb parse  1,472    0  OK
    52kb parse  1,160    0  OK
    Benchmark xeno-memory-bench: FINISH
    Benchmark xeno-speed-bench: RUNNING...
    benchmarking 4KB/hexml
    time                 6.149 μs   (6.125 μs .. 6.183 μs)
    benchmarking 4KB/xeno
    time                 2.691 μs   (2.665 μs .. 2.712 μs)

to

    Case          Bytes  GCs  Check
    4kb parse    26,096    0  OK
    42kb parse   65,696    0  OK
    52kb parse  102,128    0  OK
    Benchmark xeno-memory-bench: FINISH
    Benchmark xeno-speed-bench: RUNNING...
    benchmarking 4KB/hexml
    time                 6.225 μs   (6.178 μs .. 6.269 μs)
    benchmarking 4KB/xeno
    time                 10.34 μs   (10.06 μs .. 10.59 μs)

The first thing that should jump out at you is the allocations. What's
going on there? I looked in the profiler output, by running `stack
bench --profile` to see a profile output.

```
	Wed Jan 11 17:41 2017 Time and Allocation Profiling Report  (Final)

	   xeno-speed-bench +RTS -N -p -RTS 4KB/xeno

	total time  =        8.09 secs   (8085 ticks @ 1000 us, 1 processor)
	total alloc = 6,075,628,752 bytes  (excludes profiling overheads)

COST CENTRE            MODULE                             %time %alloc

parse.findTagName      Xeno                                35.8   72.7
getOverhead            Criterion.Monad                     13.6    0.0
parse.checkOpenComment Xeno                                 9.9    0.0
parse.findLT           Xeno                                 8.9    0.0
parse                  Xeno                                 8.4    0.0
>>=                    Data.Vector.Fusion.Util              4.6    7.7
getGCStats             Criterion.Measurement                2.8    0.0
basicUnsafeIndexM      Data.Vector.Primitive                1.6    2.0
fmap                   Data.Vector.Fusion.Stream.Monadic    1.3    2.2
rSquare.p              Statistics.Regression                1.3    1.5
basicUnsafeWrite       Data.Vector.Primitive.Mutable        1.2    1.4
innerProduct.\         Statistics.Matrix.Algorithms         1.0    1.6
qr.\.\                 Statistics.Matrix.Algorithms         0.8    1.2
basicUnsafeSlice       Data.Vector.Primitive.Mutable        0.5    1.1
transpose              Statistics.Matrix                    0.5    1.3
```

Right at the top, we have `findTagName`, doing all the allocations. So
I looked at the code, and found that the only possible thing that
could be allocating, is `S.drop`. This function skips n elements at
the start of a `ByteString`. It turns out that `S.head (S.drop index0
str)` was allocating an intermediate string, just to get the first
character of that string. It wasn't _copying_ the whole string, but it
was making a new pointer to it.

So I realised that I could just replace `S.head (S.drop n s)` with
`S.index s n`:

``` haskell
-          if S.head this == closeTagChar
+          if S.index str spaceOrCloseTag == closeTagChar
             then findLT spaceOrCloseTag
-            else if S.head this == spaceChar
+            else if S.index str spaceOrCloseTag == spaceChar
                    then findGT spaceOrCloseTag
                    else error "Expecting space or closing '>' after tag name."
-          where this = S.drop spaceOrCloseTag str
       where
         index =
-          if S.head (S.drop index0 str) == questionChar ||
-             S.head (S.drop index0 str) == slashChar
+          if S.index str index0 == questionChar ||
+             S.index str index0 == slashChar
```

And sure enough, the allocations disappeared:

    Case        Bytes  GCs  Check
    4kb parse   1,160    0  OK
    42kb parse  1,160    0  OK
    52kb parse  1,472    0  OK
    Benchmark xeno-memory-bench: FINISH
    Benchmark xeno-speed-bench: RUNNING...
    benchmarking 4KB/hexml
    time                 6.190 μs   (6.159 μs .. 6.230 μs)
    benchmarking 4KB/xeno
    time                 4.215 μs   (4.175 μs .. 4.247 μs)

Down to 4.215 μs. That's not as fast as our pre-name-parsing 2.691
μs. But we had to pay _something_ for the extra operations per
tag. We're just not allocating anymore, which is great.

## SAX for free

Eventually I ended up with a function called `process` that parses XML
and triggers events in a SAX style:

``` haskell
process
  :: Monad m
  => (ByteString -> m ())               -- ^ Open tag.
  -> (ByteString -> ByteString -> m ()) -- ^ Tag attribute.
  -> (ByteString -> m ())               -- ^ End open tag.
  -> (ByteString -> m ())               -- ^ Text.
  -> (ByteString -> m ())               -- ^ Close tag.
  -> ByteString -> m ()
```

Thanks again to GHC's optimizations, calling this function purely and
doing nothing is exactly equal to the function before SAX-ization:

``` haskell
-- | Parse the XML but return no result, process no events.
validate :: ByteString -> Bool
validate s =
  case spork
         (runIdentity
            (process
               (\_ -> pure ())
               (\_ _ -> pure ())
               (\_ -> pure ())
               (\_ -> pure ())
               (\_ -> pure ())
               s)) of
    Left (_ :: XenoException) -> False
    Right _ -> True
```

    Case        Bytes  GCs  Check
    4kb parse   1,472    0  OK
    42kb parse  1,160    0  OK
    52kb parse  1,472    0  OK

    benchmarking 4KB/xeno
    time                 4.320 μs   (4.282 μs .. 4.361 μs)

This function performs at the same speed as `process` before it
accepted any callback arguments. This means that the only overhead to
SAX'ing will be the activities that the callback functions themselves
do.

## Specialization is for insects (and, as it happens, optimized programs)

One point of interest is that adding a `SPECIALIZE` pragma for the
`process` function increases speed by roughly 1 μs. Specialization
means that for a given function which is generic (type-class
polymorphic), which means it will accept a dictionary argument at
runtime for the particular instance, instead we will generate a
separate piece of code that is specialized on that exact
instance. Below is the `Identity` monad's (i.e. just pure, does
nothing) specialized type for `process`.

``` haskell
{-# SPECIALISE
    process
      :: (ByteString -> Identity ())
      -> (ByteString -> ByteString -> Identity ())
      -> (ByteString -> Identity ())
      -> (ByteString -> Identity ())
      -> (ByteString -> Identity ())
      -> ByteString
      -> Identity ()
 #-}
```

Before

    benchmarking 4KB/xeno-sax
    time                 5.877 μs   (5.837 μs .. 5.926 μs)
    benchmarking 211KB/xeno-sax
    time                 285.8 μs   (284.7 μs .. 287.4 μs)

after

    benchmarking 4KB/xeno-sax
    time                 5.046 μs   (5.036 μs .. 5.056 μs)
    benchmarking 211KB/xeno-sax
    time                 240.6 μs   (240.0 μs .. 241.5 μs)


In the 4KB case it's only 800 ns, but as we say in Britain, take care
of the pennies and the pounds will look after themselves. The 240->285
difference isn't big in practical terms, but when we're playing the
speed game, we pay attention to things like that.

## Where we stand: Xeno vs Hexml

Currently the SAX interface in Zeno outperforms Hexml in space and
time. Hurrah! We're as fast as C!

    File   hexml-dom  xeno-sax
    4KB    6.134 μs   5.147 μs
    31KB   9.299 μs   2.879 μs
    211KB  257.3 μs   241.0 μs

It's also worth noting that Haskell does this all safely. All the
functions I'm using are standard ByteString functions which do bounds
checking and throw an exception if so. We don't accidentally access
memory that we shouldn't, and we don't segfault. The server keeps
running.

If you're interested, if we switch to unsafe functions (`unsafeTake`,
`unsafeIndex` from the `Data.ByteString.Unsafe` module), we get a
notable speed increase:

    File   hexml-dom  xeno-sax
    4KB    6.134 μs   4.344 μs
    31KB   9.299 μs   2.570 μs
    211KB  257.3 μs   206.9 μs

We don't need to show off, though. We've already made our point. We're
Haskellers, we like safety. I'll keep my safe functions.

## But Hexml does more!

I'd be remiss if I didn't address the fact that Hexml does more useful
things than we've done here. Hexml allocates a DOM for random
access. Oh no! Allocation: Haskell's worse enemy!

[image deleted due to region politics]

We've seen that Haskell allocates a lot normally. Actually, have we
looked at that properly?

    Case                   Bytes  GCs  Check
    4kb/hexpat-sax       444,176    0  OK
    31kb/hexpat-sax      492,576    0  OK
    211kb/hexpat-sax  21,112,392   40  OK
    4kb/hexpat-dom       519,128    0  OK
    31kb/hexpat-dom      575,232    0  OK
    211kb/hexpat-dom  23,182,560   44  OK

Alright.

## Implementing a DOM parser for Xeno

All isn't lost. Hexml isn't a dumb parser that's fast because it's in
C, it's also a decent algorithm. Rather than allocating a tree, it
allocates a big flat vector of nodes and attributes, which contain
offsets into the original string. We can do that in Haskell too!

Here's my design of a data structure contained in a vector. We want to
store just integers in the vector. Integers that point to offsets in
the original string. Here's what I came up with.

We have three kinds of payloads. Elements, text and attributes:

``` ruby
1.  00 # Type tag: element
2.  00 # Parent index (within this array)
3.  01 # Start of the tag name in the original string
4.  01 # Length of the tag name
5.  05 # End index of the tag (within this array)
```

``` ruby
1.  02 # Type tag: attribute
2.  01 # Start of the key
3.  05 # Length of the key
4.  06 # Start of the value
5.  03 # Length of the value
```

``` ruby
1.  01 # Type tag: text
2.  01 # Start of the text
3.  10 # Length of the text
```

That's all the detail I'm going to go into. You can read
[the code](https://github.com/chrisdone/xeno) if you want to know
more. It's not a highly optimized format. Once we have such a vector,
it's possible to define a DOM API on top of it which can let you
navigate the tree as usual, which we'll see later.

We're going to use our SAX parser--the `process` function, and we're
going to implement a function that writes to a big array. This is a
very imperative algorithm. Haskellers don't like imperative algorithms
much, but Haskell's fine with them.

The function ends up looking something like this:

``` haskell
runST
  (do nil <- UMV.new 1000
      vecRef <- newSTRef nil
      sizeRef <- fmap asURef (newRef 0)
      parentRef <- fmap asURef (newRef 0)
      process
        (\(PS _ name_start name_len) ->
            <write the open tag elements>)
        (\(PS _ key_start key_len) (PS _ value_start value_len) ->
            <write an attribute into the vector>)
        (\_ -> <ignore>)
        (\(PS _ text_start text_len) ->
            <write a text entry into the vector>)
        (\_ ->
            <set the end position of the parent>
            <set the current element to the parent>)
        str
      wet <- readSTRef vecRef
      arr <- UV.unsafeFreeze wet
      size <- readRef sizeRef
      return (UV.unsafeSlice 0 size arr))
```

The function runs in the `ST` monad which lets us locally read and
write to mutable variables and vectors, while staying pure on the
outside.

I allocate an array of 1000 64-bit Ints (on 64-bit arch), I keep a
variable of the current size, and the current parent (if any). The
current parent variable lets us, upon seeing a </close> tag, assign
the position in the vector of where the parent is closed.

Whenever we get an event and the array is too small, I grow the array
by doubling its size. This strategy is copied from the Hexml package.

Finally, when we're done, we get the mutable vector, "freeze" it (this
means making an immutable version of it), and then return that
copy. We use `unsafeFreeze` to re-use the array without copying, which
includes a promise that we don't use the mutable vector afterwards,
which we don't.

## The DOM speed

Let's take a look at the speeds:

    File   hexml-dom  xeno-sax  xeno-dom
    4KB    6.123 μs   5.038 μs  10.35 μs
    31KB   9.417 μs   2.875 μs  5.714 μs
    211KB  256.3 μs   240.4 μs  514.2 μs

Not bad! The DOM parser is only <2x slower than Hexml (except in the
31KB where it's faster. _shrug_). Here is where I stopped optimizing
and decided it was good enough. But we can review some of the
decisions made along the way.

In the code we're using unboxed mutable references for the current
size and parent, the mutable references are provided by the
mutable-containers package. See these two lines here:

``` haskell
      sizeRef <- fmap asURef (newRef 0)
      parentRef <- fmap asURef (newRef 0)
```

Originally, I had tried `STRef`'s, which are boxed. Boxed just means
it's a pointer to an integer instead of an actual integer. An unboxed
Int is a proper machine register. Using an `STRef`, we get worse
speeds:

    File   xeno-dom
    4KB    12.18 μs
    31KB   6.412 μs
    211KB  631.1 μs

Which is a noticeable speed loss.

Another thing to take into consideration is the array type. I'm using
the unboxed mutable vectors from the `vector` package. When using
atomic types like Int, it can be a leg-up to use unboxed
vectors. If I use the regular boxed vectors from `Data.Vector`, the
speed regresses to:

    File   xeno-dom
    4KB    11.95 μs (from 10.35 μs)
    31KB   6.430 μs (from 5.714 μs)
    211KB  1.402 ms (from 514.2 μs)

Aside from taking a bit more time to do writes, it also allocates 1.5x
more stuff:

    Case                 Bytes  GCs  Check
    4kb/xeno/dom        11,240    0  OK
    31kb/xeno/dom       10,232    0  OK
    211kb/xeno/dom   1,082,696    0  OK

becomes

    Case                 Bytes  GCs  Check
    4kb/xeno/dom        22,816    0  OK
    31kb/xeno/dom       14,968    0  OK
    211kb/xeno/dom   1,638,392    1  OK

See that GC there? We shouldn't need it.

Finally, one more remark for the DOM parser. If we forsake safety and
use the `unsafeWrite` and `unsafeRead` methods from the vector
package, we do see a small increase:

    File   xeno-dom
    4KB    9.827 μs
    31KB   5.545 μs
    211KB  490.1 μs

But it's nothing to write home about. I'll prefer memory safety over a
few microseconds this time.

## The DOM API

I wrote some functions to access our vector and provide a DOM-like
API:

``` haskell
> let Right node = parse "<foo k='123'><p>hi</p>ok</foo>"
> node
(Node "foo" [("k","123")] [Element (Node "p" [] [Text "hi"]),Text "ok"])
> name node
"foo"
> children node
[(Node "p" [] [Text "hi"])]
> attributes node
[("k","123")]
> contents node
[Element (Node "p" [] [Text "hi"]),Text "ok"]
```

So that works.

## Wrapping-up

The final results are in:

[image deleted due to region politics]

And just to check that a 1MB file doesn't give wildly different
results:

    benchmarking 1MB/hexml-dom
    time                 1.225 ms   (1.221 ms .. 1.229 ms)
                         1.000 R²   (1.000 R² .. 1.000 R²)
    mean                 1.239 ms   (1.234 ms .. 1.249 ms)
    std dev              25.23 μs   (12.28 μs .. 40.84 μs)

    benchmarking 1MB/xeno-sax
    time                 1.206 ms   (1.203 ms .. 1.211 ms)
                         1.000 R²   (1.000 R² .. 1.000 R²)
    mean                 1.213 ms   (1.210 ms .. 1.218 ms)
    std dev              14.58 μs   (10.18 μs .. 21.34 μs)

    benchmarking 1MB/xeno-dom
    time                 2.768 ms   (2.756 ms .. 2.779 ms)
                         1.000 R²   (1.000 R² .. 1.000 R²)
    mean                 2.801 ms   (2.791 ms .. 2.816 ms)
    std dev              41.10 μs   (30.14 μs .. 62.60 μs)

Tada! We matched Hexml, in pure Haskell, using safe accessor
functions. We provided a SAX API which is very fast, and a simple
demonstration DOM parser with a familiar API which is also quite
fast. We use reasonably little memory in doing so.

This package is an experiment for educational purposes, to show what
Haskell can do and what it can't, for a very specific domain
problem. If you would like to use this package, consider adopting it
and giving it a good home. I'm not looking for more packages to
maintain.
