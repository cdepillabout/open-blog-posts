------------------------------------------------------
title: Getting Started with PureNix
summary: How to setup an environment for playing around with PureNix
tags: haskell, nixos, purescript
draft: false
------------------------------------------------------

*This post is the first post in a
[series about PureNix](./2021-12-26-purenix).*

[PureNix](https://github.com/purenix-org/purenix) is a Nix backend for
[PureScript](https://www.purescript.org/).  PureNix allows you to write
PureScript code and transpile it to Nix code.

This post gives an overview of PureScript, PureNix, and explains how to get
started using PureNix.

## PureScript

[PureScript](https://en.wikipedia.org/wiki/PureScript) is a strongly-typed,
pure functional programming language. Here's a small example of a function
written in PureScript.  If you know Haskell, this should look very familiar:

```purescript
module Main where

import Prelude

type FilePath = String

subdirectory :: FilePath -> FilePath -> FilePath
subdirectory p1 p2 = p1 <> "/" <> p2
```

The PureScript compiler compiles code like this to JavaScript.  PureScript ends
up being a Haskell-like language that you can use to write frontend code.

One of the main build tools for PureScript is
[Spago](https://github.com/purescript/spago).  Spago is somewhat similar to
Haskell's `stack` or `cabal`, Rust's `cargo`, etc.

One interesting feature of the PureScript compiler and Spago is that there is
good support for switching out the backend code generator.  The code generator
built into the main PureScript compiler is for JavaScript, but there are
alternative code generators for other languages.  PureNix is an alternative
backend code generator for Nix.

## PureNix

PureNix works together with PureScript and Spago to compile PureScript code
into Nix code.  Here's a small example of PureScript code, and what it gets
compiled to:

```purescript
module Main where

greeting :: String
greeting = "Hello, world!"

identity :: forall a. a -> a
identity a = a

data Maybe a = Nothing | Just a

fromMaybe :: forall a. a -> Maybe a -> a
fromMaybe a Nothing = a
fromMaybe _ (Just a) = a
```

PureNix compiles this PureScript code to the following Nix code[^1]:

```nix
let
  greeting = "Hello, world!";
  Nothing = {__tag = "Nothing";};
  Just = value0:
    { __tag = "Just";
      __field0 = value0;
    };
  fromMaybe = v: v1:
    let
      __pattern0 = __fail: if v1.__tag == "Nothing" then let a = v; in a else __fail;
      __pattern1 = __fail: if v1.__tag == "Just" then let a = v1.__field0; in a else __fail;
      __patternFail = builtins.throw "Pattern match failure in src/Main.purs at 11:1 - 11:41";
    in
      __pattern0 (__pattern1 __patternFail);
  identity = v: v;
in
  {inherit greeting Nothing Just fromMaybe;}
```

Except for the pattern matching in `fromMaybe`, this should look pretty
straight-forward.  You can see the PureScript string `greeting` becomes a
simple Nix string.  `identity` has become a Nix function.  The constructors for
`Maybe` (`Nothing` and `Just`) have become Nix functions[^2].  `fromMaybe` has
become a Nix function.  If you work through the `__patternX` continuations in
`fromMaybe`, you can see how they emulate the pattern matching in the PureScript
`fromMaybe` function.

Now that you have a taste of PureNix, the next section explains how to get
started actually writing PureNix code.

## Getting Started

There is a post on the NixOS Discourse that explains how to
[put together a development environment for PureNix](https://discourse.nixos.org/t/purenix-nix-backend-for-purescript/15756/3).
It explains how to get to the point where you can actually start writing
PureScript and compiling it to Nix.

Once you have your environment setup, you'll likely be interested in what
libraries are available for PureNix.  The currently available libraries
are in the [purenix-org](https://github.com/purenix-org/) organization on
GitHub.  There is also two PureNix issues you may be interested:

-   A [tracking issue](https://github.com/purenix-org/purenix/issues/37)
    for which PureScript standard libraries still need to be ported to PureNix.
    If you're looking for a way to help out PureNix, this is a good place
    to start!
-   An issue for putting together a real
    [PureNix package set](https://github.com/purenix-org/purenix/issues/36).
    A PureScript/PureNix package set is similar to a Stackage resolver.  It is
    used by Spago to make it easier to find library versions that work together.

You may also be interested in [Pursuit](https://pursuit.purescript.org/).
This is like Hoogle, but for PureScript.  Pursuit doesn't know anything
about PureNix, but the API of the PureNix libraries that have been ported so
far is almost the same as the PureScript version of the libraries, so searching
for functions on Pursuit is still helpful.

Now that you have an idea how to get started with PureNix, what sorts of
developers will find PureNix easy to use?

## Who is PureNix for?

If you have experience with PureScript, Nix, and Haskell, PureNix will be easy
to get started with.  You may have to write some FFI for Nix builtins, but
other than that you should have no trouble using PureNix.  It should feel
quite natural.

If you have experience with PureScript and Nix (but not experience with
Haskell), PureNix should still be pretty easy.  Unlike the main JavaScript
backend for PureScript, PureNix is lazy.  This will feel a little weird at
first, but since you already know Nix, this shouldn't be too difficult.
There are also a few type-classes in PureScript that aren't necessary in
PureNix, like
[Lazy](https://pursuit.purescript.org/packages/purescript-control/5.0.0/docs/Control.Lazy#t:Lazy)
and
[MonadRec](https://pursuit.purescript.org/packages/purescript-tailrec/5.0.1/docs/Control.Monad.Rec.Class).

If you have experience with Haskell and Nix (but not PureScript), it
might take you a little bit of time to get used to the PureScript
standard library, but other than that you shouldn't have many problems.
In general, PureScript is split up into a large number of small packages and
modules, more so than Haskell.  But most functions are named similarly, so
you shouldn't have much trouble in practice.  You'll likely find the
following resources very useful:

-   [Pursuit](https://pursuit.purescript.org/).  A combination of a search engine
    like Hoogle, and API documentation like Hackage, but for PureScript.
-   [Differences from Haskell](https://github.com/purescript/documentation/blob/master/language/Differences-from-Haskell.md).
    Official documentation on the differences between PureScript and Haskell.
-   [Language Documentation](https://github.com/purescript/documentation/tree/master/language).
    Official documentation on the PureScript language.  You'll likely want to
    focus on the document about Records, since they are one of the main
    differences between PureScript and Haskell.

If you have experience with Haskell or PureScript (but not Nix), you may
have some trouble getting started.  It may be a little early to attempt
to use PureNix.  You'd likely benefit from first reading the
[Nix Manual](https://nixos.org/manual/nix/stable/).

If you have experience with Nix (but not Haskell or PureScript), you will
likely have trouble getting started with PureNix.  I would recommend
first trying to learn Haskell or PureScript.  If your goal is to use PureNix,
then it doesn't really matter if you first learn Haskell or PureScript, since
they are so similar.  Haskell seems to have more beginner-oriented resources
than PureScript.  Keep in mind that like Nix, Haskell and PureScript have a
steep learning curve.  Becoming proficient in Haskell or PureScript can be
quite time-consuming.

## Conclusion

*This post is the first post in a
[series about PureNix](./2021-12-26-purenix).*

## Footnotes

[^1]: This has been slightly simplified to make it a little easier to understand.

[^2]: PureScript data constructors become tagged records in Nix.
