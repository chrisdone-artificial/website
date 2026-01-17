---
date: 2014-11-20
title: "Lucid: templating DSL for HTML"
description: "Lucid: templating DSL for HTML"
author: Chris Done
tags: haskell
---

I'm not big on custom templating languages, for reasons I'll write
about another time. I prefer EDSLs. I preferred the
[xhtml](http://hackage.haskell.org/package/xhtml) package back when
that was what everybody used. It looked like this:

``` haskell
header << thetitle << "Page title"

thediv noHtml ! [theclass "logo"] << "…"
thediv noHtml ! [identifier "login"]
```

Pretty line-noisy to read, write and hard to edit in a reasonable
manner.

Later, [blaze-html](http://hackage.haskell.org/package/blaze-html)
became the new goto HTML writing library. It improved upon the XHTML
package by being faster and having a convenient monad instance. It
looks like this:

``` haskell
page1 = html $ do
    head $ do
        title "Introduction page."
        link ! rel "stylesheet" ! type_ "text/css" ! href "screen.css"
    body $ do
        div ! id "header" $ "Syntax"
        p "This is an example of BlazeMarkup syntax."
        ul $ mapM_ (li . toMarkup . show) [1, 2, 3]
```

Much easier to read, write and edit thanks to the monad instance.

However, after several years of using that, I've come to write my
own. I'll cover the infelicities about Blaze and then discuss my
alternative approach.

Reading back through what I've written below, it could be read as a
bit attacky, and some of the issues are less philosophical and more
incidental. I think of it more that the work on writing HTML in a DSL
is incomplete and to some degree people somewhat gave up on doing it
more conveniently at some point. So I'm re-igniting that.

The combination of having a need to write a few HTML reports and
recent discussions about Blaze made me realise it was time for me to
come at this problem a-fresh with my own tastes in mind. I also
haven't used my own approach much, other than porting some trivial
apps to it.

# Blaze

## Names that conflict with base

The first problem is that Blaze exports many names which conflict with
base. Examples:

`div`, `id`, `head`, `map`

The obvious problem with this is that you either have to qualify any
use of those names, which means you have to qualify Blaze, and end up
with something inconsistent like this:

``` haskell
H.div ! A.id "logo" $ "…"
```

Where `H` and `A` come from importing the element and attribute
modules like this:

``` haskell
import qualified Text.Blaze.Html5            as H
import qualified Text.Blaze.Html5.Attributes as A
```

Or you don't import `Prelude` and only import Blaze, but then you
can't do a simple `map` without qualification.

You might've noticed in the old `xhtml` package that `thediv` and
`identifier` are used instead. The problem with using different names
from the actual things they refer to is that they're hard to learn and
remember, both for regular Haskellers and newbies coming to edit your
templates.

## Names that are keywords

This is a common problem in DSLs, too. In Blaze the problem is:
`class` or `type` (perhaps others I don't recall). Blaze solves it
with: `class_` or `type_`

Again, the problem with this is that it is inconsistent with the other
naming conventions. It's another exception to the rule that you have
to remember and makes the code look bad.

## Conflicting attribute and element names

There are also names which are used for both attributes and
elements. Examples are `style` and `map`. That means you can't write:

``` haskell
H.head $ style "body { background: red; }"
body $ p ! style "foo" $ …
```

You end up writing:

``` haskell
H.head $ H.style "body { background: red; }"
body $ p ! A.style "foo" $ …
```

## Inconsistency is difficult and ugly

What the above problems amount to is ending up with code like this:

``` haskell
body $ H.div ! A.id "logo" ! class_ "left" ! hidden $ "Content"
```

At this point users of Blaze give up with second-guessing every markup
term they write and decide it's more consistent to qualify
_everything_:

``` haskell
H.body $ H.div ! A.id "logo" ! A.class_ "left" ! A.hidden $ "Content"
```

Or, taken from some real code online:

``` haskell
H.input H.! A.type_ "checkbox"
        H.! A.checked True
        H.! A.readonly "true"
```

This ends up being too much. Inconvenient to type, ugly to read, and
one more step removed from the HTML we're supposed to be generating.

## The Monad instance isn't

The monad instance was originally conceived as a handy way to write
HTML nicely without having to use `<>` or lists of lists and other
less wieldy syntax.

In the end the monad ended up being defined like this:

``` haskell
instance Monad MarkupM where
  return _ = Empty
  {-# INLINE return #-}
  (>>) = Append
  {-# INLINE (>>) #-}
  h1 >>= f = h1 >> f
      (error "Text.Blaze.Internal.MarkupM: invalid use of monadic bind")
  {-# INLINE (>>=) #-}
```

And has been for some years. Let's take a trivial example of why this
is not good. You render some HTML and while doing so build a result to
be used later:

``` haskell
do xs <- foldM (\c i -> …)
               mempty
               ys
   mapM_ dd xs
```

Uh-oh:

    *** Exception: Text.Blaze.Internal.MarkupM: invalid use of monadic bind

## The overloaded strings instance is bad

The previous point leads onto this next point, which is that due to
this phantomesque monad type, the instance is like this:

``` haskell
instance IsString (MarkupM a) where
    fromString = Content . fromString
    {-# INLINE fromString #-}
```

How can it make this value? It cannot. If you want to go ahead and
extract that `a', you get:

    *** Exception: Text.Blaze.Internal.MarkupM: invalid use of monadic bind

Additionally, this instance is too liberal. You end up getting this
warning:

> A do-notation statement discarded a result of type `GHC.Prim.Any`
>
> Suppress this warning by saying `_ <- "Example"`
> or by using the flag `-fno-warn-unused-do-bind`

So you end up having to write in practice (again, taken from a real
Blaze codebase by one of the authors):

    void "Hello!"

Which pretty much negates the point of using `IsString` in the
first-place. Alternatively, you use `-fno-warn-unused-do-bind` in your
module.

## Working with attributes is awkward

The ! syntax seems pretty convenient from superficial inspection:

``` haskell
link ! rel "stylesheet" ! type_ "text/css" ! href "screen.css"
```

But in practice it means you always have the same combination:

``` haskell
div ! H.class_ "logo" $ "…"
```

Which I find—personally speaking—a bit distasteful to read, it's not
far from what we saw in the old xhtml package:

``` haskell
thediv ! [theclass "logo"] << "…"
```

Did we really save that much in the attribute department? Operators
are evil.

But mostly presents an editing challenge. Operators like this make it
tricky to navigate, format in a regular way and do code
transformations on. All Haskell code has operators, so this is a
general problem. But if your DSL doesn't actually need these
operators, I consider this a smell.

## Attributes don't compose

You should be able to compose `with`. For example, let's say you want to
define a re-usable component with bootstrap:

``` haskell
container inner = div ! class_ "container" $ inner
```

Now you can use it to make a container. But consider now that you also
want to add additional attributes to it later. You can do that with
another call to with:

``` haskell
container ! class_ "main" $ "zot"
```

In Blaze this produces:

``` haskell
λ> main
"<div class=\"container\" class=\"main\">My content!</div>"
```

Browsers ignore the latter main, so the composition didn't work.

## Ceremony is tiring

Here's the example from Blaze's package, that's introduced to users.

``` haskell
import Prelude hiding (head, id, div)
import Text.Blaze.Html4.Strict hiding (map)
import Text.Blaze.Html4.Strict.Attributes hiding (title)
import Text.Blaze.Renderer.Utf8 (renderMarkup)

page1 :: Markup
page1 = html $ do
    head $ do
        title "Introduction page."
        link ! rel "stylesheet" ! type_ "text/css" ! href "screen.css"
    body $ do
        div ! id "header" $ "Syntax"
        p "This is an example of BlazeMarkup syntax."
        ul $ mapM_ (li . toMarkup . show) [1, 2, 3]

main = print (renderMarkup page1)
```

Apart from the import backflips you have to do to resolve names
properly, you have at least three imports to make just to render some
HTML. Call me lazy, or stupid, but I never remember this deep
hierarchy of modules and always have to look it up every single
time. And I've been using Blaze for as long as the authors have.

## Transforming

A smaller complaint is that it would sometimes be nice to transform
over another monad. Simplest example is storing the read-only model
information in a reader monad and then you don't have to pass around a
bunch of things as arguments to all your view functions. I'm a big fan
of function arguments for explicit state, but not so much if it's the
same argument every time.

## No Show instance

It would be nice if you could just write some markup in the REPL
without having to import some other modules and wrap it all in a
function just to see it.

# Lucid

My new library, Lucid, attempts to solve most of these problems.

## Naming issues

Firstly, *all* names which are representations of HTML terms are
suffixed with an underscore `_`:

    p_, class_, table_, style_

No ifs or buts. *All markup terms.*

That solves the following problems (from the issues described above):

* Names that conflict with base: `div_`, `id_`, `head_`, `map_`, etc.
* Names that are keywords: `class_`, `type_`, etc.
* Conflicting attribute and element names: solved by abstracting those
  names via a class. You can write `style_` to mean either the element
  name or the attribute name.
* Inconsistency is difficult and ugly: there's no inconsistency, all
  names are the same format.

No import problems or qualification. Just write code without worrying
about it.

## How it looks

Plain text is written using the `OverloadedStrings` and
`ExtendedDefaultRules` extensions, and is automatically escaped:

``` haskell
λ> "123 < 456" :: Html ()
123 &lt; 456
```

Elements nest by function application:

``` haskell
λ> table_ (tr_ (td_ (p_ "Hello, World!")))
<table><tr><td><p>Hello, World!</p></td></tr></table>
```

Elements are juxtaposed via monoidal append:

``` haskell
λ> p_ "hello" <> p_ "sup"
<p>hello</p><p>sup</p>
```

Or monadic sequencing:

``` haskell
λ> div_ (do p_ "hello"; p_ "sup")
<div><p>hello</p><p>sup</p></div>
```

Attributes are set using the with combinator:

``` haskell
λ> with p_ [class_ "brand"] "Lucid Inc"
<p class="brand">Lucid Inc</p>
```

Conflicting attributes (like `style_`) work for attributes or
elements:

``` haskell
λ> html_ (head_ (style_ "body{background:red}") <>
                 with body_ [style_ "color:white"]
                      "Look ma, no qualification!")
<html><head><style>body{background:red}</style></head>
<body style="color:white">Look ma, no qualification!</body></html>
```

## The Blaze example

For comparison, here's the Blaze example again:

``` haskell
page1 = html $ do
    head $ do
        title "Introduction page."
        link ! rel "stylesheet" ! type_ "text/css" ! href "screen.css"
    body $ do
        div ! id "header" $ "Syntax"
        p "This is an example of BlazeMarkup syntax."
        ul $ mapM_ (li . toMarkup . show) [1, 2, 3]
```

And the same thing in Lucid:

``` haskell
page2 = html_ $ do
    head_ $ do
      title_ "Introduction page."
      with link_ [rel_ "stylesheet",type_ "text/css",href_ "screen.css"]
    body_ $ do
        with div_ [id_ "header"] "Syntax"
        p_ "This is an example of Lucid syntax."
        ul_ $ mapM_ (li_ . toHtml . show) [1,2,3]
```

I'm not into operators like `($)` and swung indentation like that, but
I followed the same format.

I'd write it in a more Lispy style and run my hindent tool on it:

``` haskell
page1 =
  html_ (do head_ (do title_ "Introduction page."
                      with link_
                           [rel_ "stylesheet"
                           ,type_ "text/css"
                           ,href_ "screen.css"])
            body_ (do with div_ [id_ "header"] "Syntax"
                      p_ "This is an example of Lucid syntax."
                      ul_ (mapM_ (li_ . toHtml . show)
                                 [1,2,3])))
```

But that's another discussion.

## It's a real monad

Normal monadic operations work properly:

``` haskell
λ> (return "OK!" >>= p_)
<p>OK!</p>
```

It's basically a writer monad.

In fact, it's also a monad transformer:

``` haskell
λ> runReader (renderTextT (html_ (body_ (do name <- lift ask
                                            p_ (toHtml name)))))
             ("Chris" :: String)
"<html><body><p>Chris</p></body></html>"
```

## Overloaded strings instance is fine

The instance is constrained over the return type being `()`. So string
literals can only be type `HtmlT m ()`.

``` haskell
λ> do "x" >> "y" :: Html ()
xy

λ> do x <- "x"; toHtml (show x)
x()
```

## Attributes

Attributes are simply written as a list. That's all. Easy to
manipulate as a data structure, easy to write and edit, and
automatically indent in a predictable way:

``` haskell
λ> with p_ [id_ "person-name",class_ "attribute"] "Mary"
<p id="person-name" class="attribute">Mary</p>
```

No custom operators are required. Just the `with` combinator. If you
want to indent it, just indent it like normal function application:

``` haskell
with p_
     [id_ "person-name",class_ "attribute"]
     "Mary"
```

And you're done.

## Composing attributes

You should be able to compose `with`. For example, let's say you want to
define a re-usable component with bootstrap:

``` haskell
λ> let container_ = with div_ [class_ "container "]
```

Now you can use it to make a container:

``` haskell
λ> container_ "My content!"
<div class="container ">My content!</div>
```

But consider now that you also want to add additional attributes to it
later. You can do that with another call to with:

``` haskell
λ> with container_ [class_ "main"] "My content!"
<div class="container main">My content!</div>
```

Duplicate attributes are composed with normal monoidal append. Note
that I added a space in my definition of container anticipating
further extension later. Other attributes might not compose with
spaces.

## Unceremonious

Another part I made sure was right was lack of import nightmare. You
just `import Lucid` and away you go:

``` haskell
λ> import Lucid
λ> p_ "OK!"
<p>OK!</p>
λ> p_ (span_ (strong_ "Woot!"))
<p><span><strong>Woot!</strong></span></p>
λ> renderBS (p_ (span_ (strong_ "Woot!")))
"<p><span><strong>Woot!</strong></span></p>"
λ> renderToFile "/tmp/foo.html" (p_ (span_ (strong_ "Woot!")))
```

If I want to do more advanced stuff, it's all available in
`Lucid`. But by default it's absolutely trivial to get going and
output something.

## Speed

Actually, despite having a trivial implementation, being a real monad
and a monad transformer, it's not far from Blaze. A quick test of
writing 38M of HTML to file yielded the same speed (about 1.5s) for
both Lucid and Blaze. With such decent performance for very little
work I'm already ready to start using it for real work.

## Summary

So the point of this post was really to explain *why another HTML DSL*
and I hope I did that well enough.

The code is [on Github](https://github.com/chrisdone/lucid). I pushed
to Hackage but you can consider it beta for now.
