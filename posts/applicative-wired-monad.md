---
title: Applicative-wired monad pattern
date: 2025-07-11
description: Different ways of writing code
---

In Haskell API design, you sometimes want to model a computation that looks like a monad, 
i.e. some things depend on other things, and make use of do-notation, 
but you want to be able to statically inspect the resulting structure, too. 

The ApplicativeDo notation attempts to bridge this gap by a language extension and some conventions. 
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

The Action is some instance of Monad, and could be a free monad. 
The Value is some instance of Applicative, and can be a free applicative.

The trick is that all functions exposed by the API only return the type Action (Value a), 
and sometimes accept Value a as arguments. This means you wire up a graph, 
with Action containing nodes and Value serving the edges. 
You combine multiple output values into a single argument value via its Applicative instance. 
Then it’s easy to either run it as a regular action (by interpreting Value as Identity), 
or graph it out or batch it as needed.  Works for SQL DBs (e.g. Rel8), build systems or FRP (e.g. Reflex).

This does mean you can’t simply run mapM against a Value [a], 
and this often requires a special operator for the action in the domain in question.

I’m not sure that there’s already name for it, but it’s definitely a pattern. 
You see it in quite a few places. So I'm pointing it out.

I'll add a code example at a later date.
