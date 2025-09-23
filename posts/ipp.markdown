---
date: 2022-02-11
title: Immutable Publishing Policy
description: Immutable Publishing Policy
author: Chris Done
tags: haskell
---

## Summary

This is a policy for publishing Haskell packages. It was published 
in **February of 2022**. The aim of this policy is to remove all
breaking changes to downstream users of packages, where feasible.
It is most applicable to packages which have users, and/or have 
a more or less stable API. It prioritizes downstream users at the 
expense of author convenience.

It has been implemented in the 
[lucid](https://github.com/chrisdone/lucid)
package.

### Terms

By "levels" in this description, we mean "module" is the first level,
and "package" is the level above.

### Rules

* If you **require more**, that's a break.
  * E.g. adding an extra field to a record or requiring a class
    constraint, add an extra constructor which users can pattern match on, or another 
    package dependency or newer major version of a package that wasn't needed before.
* If you **provide less**, that's a break.
  * E.g. if you remove an export or a field of a record, return less or different
  from a function, remove a module, or an instance of an exposed type.
* If you **change something**, that's a break of the next level up.
  * E.g. if you change the type signature of an exposed function, or
    change a data type's definition which is exposed.
* If you **just add** something, then just bump the minor version (which is the only thing that changes in this scheme) of
  the package;
  you can use the current date e.g. `0.0.20210101`.
  * E.g. a non-orphan instance of a type from your own package.

Haskell-specific note:

* If you add an orphan instance in your
  library--which you **should not do**--that's definitely a breaking change
  for the whole package, **bump the package.**
* A user may have written an orphan instance for one of your types, but
  this is a departure from standard practice, and therefore isn't
  protected from breaking change. You should feel free to **add new
  instances** for your own types.
* [Be careful about leaking instances](https://pvp.haskell.org/#leaking-instances). If
  you bump your package when you bump your dependencies major versions, then you should be
  safe from this. 

### Steps

This **differs substantially to [the PVP](https://pvp.haskell.org/).** ⚠️

* If you want to change a thing `foo`, just make a copy of it and call it `foo2`.
* If it's impractical to add another version of a thing (because,
  e.g. you changed a class or a data type that's used in many places),
  then copy the whole module `Foo` to `Foo2`.
* If it's impractical to just copy a module, then it's time to copy
  the package, `foo` to `foo2`.
* If you want to rename or remove something from a module, clone the module. 
  If you want to remove or rename a module, clone the package.

### Rationale

* It's more effort for an author to make breaking changes. The author
  pays a price, and has to think twice about whether it's worth it.
* The user **optionally** pays a small price to upgrade to new
  functionality, rather than a _mandatory tax_ for changes they don't
  want, which is the PVP system.
* For more elaboration, watch Rich Hickey's video
  [Spec-ulation](https://www.youtube.com/watch?v=oyLBGkS5ICk).
* Related mantra of the Linux kernel: _[don't break user-space](https://unix.stackexchange.com/questions/235335/why-is-there-a-linux-kernel-policy-to-never-break-user-space)_
* I don't agree at all with [this post](http://blog.ezyang.com/2016/12/thoughts-about-spec-ulation-rich-hickey/) by ezyang

### Long-term steps

* You can publish additions to your old versions of modules and
  packages.
* You can choose to stop maintaining them at some point. For example,
  a duplicate module hierarchy. You can just drop the old one and
  publish a new package.
* By the PVP, packages that use your package will force their users to
  use your newer API, which will make them unhappy. By following the
  IPP, you give your users the chance to import both library versions
  and maybe write some migrations between the two versions.

### Infrastructure/technical support

* Hackage supports deprecating a package in favour of something
  else. That's your means to tell users of newer versions.
* GHC lets you mark declarations and modules as deprecated, which is
  issued as a warning.
* Users can use `module Lib (module Lib2) where import Lib2` to try
  newer versions of module hierarchies as "drop-ins".
* For a package following the IPP, you can just depend 
  on `package >= 0.0.n` where `n` is the feature that 
  you need. You never need an upper bound because there are no
  breaking changes.
* Both Hackage and Stackage will allow you to upload package1, package2, 
  etc. and so your users can continue to use the package1 if they're not
  interested in package2, and if they're not forced by PVP-following 
  maintainers of other packages that they're using.

### Compatibility with the PVP

You can do both, the IPP is a subset of behavior within the PVP (but upper bounds aren't required or needed by the IPP). In practice,
people following just the PVP and not the IPP will be a pain for you as Haskellers
change things and cause breakage all the time. But don't bother people about 
it. Just do your best.

### When to use this policy

I propose that you should:

* Put new experimental packages on GitHub/GitLab/sourcehut, etc. and using Stack or Cabal you can specify dependencies on that package by a commit hash. At this stage, authors can change as much as they want and users can pin to commits.
* If your design becomes stable and you want to support the package, put it on Stackage or Hackage. Now apply IPP.

For packages already published, you can apply IPP immediately.

### FAQs

#### Do you just want Unison?

Sure, but that's a whole new language. Haskell is our language of choice, so let's do our best to make it nicer to use.

#### It's not worth doing because we should have machines automate dependencies based on fine grained uses

That's true. Here in the real world, however, no such automation exists.

#### Security fixes can be breaking changes, we should make bad code go away

You don't know how and in what setting your users are using your libraries. It's not your place
to bully them into changing their programs because you know better.

#### Shouldn't old code be garbage collected?

Yes, I covered deprecation above. I also covered removing old code and the ending of maintenance above.

#### What if *I* add a constructor to a type? *I'd* have to duplicate my whole API!

Yes. If *you* add a new constructor then *you'll* indeed **require more** 
of the consumer of your package to form a complete pattern match. Duplicate the 
type. *You* changed a namespace.

I'll repeat myself: the IPP is about valuing the user's time over the maintainer's time. *It's not
all about you and your needs.* Think about others who have to consume your work.

Even the containers package authors plan to [remove old names](https://github.com/haskell/containers/pull/1049/files), despite there being no reason to do so.

#### Why have package versions when you are going to duplicate an entire module or package when you make a breaking change?

I already covered this above, but let's rewrite in a different way: Package versions in Haskell are broken.
They force all package maintainers to move in lock step. You can only practically use **one** version of a package in your installed package scope. 

If you want to keep using foo-1.2.3 and another package
you're using wants foo-2.0.0, you shouldn't be forced to change all your code. You should be given the option
to migrate between them.
