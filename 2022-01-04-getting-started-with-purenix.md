------------------------------------------------------
title: Getting Started with PureNix
summary: How to setup an environment for playing around with PureNix
tags: haskell, nixos, purescript
draft: false
------------------------------------------------------

*This post is the first post in a
[series about PureNix](./2022-01-03-purenix). The next post is about
[who would find PureNix easy to use](./2022-01-05-who-would-like-purenix).*

[PureNix](https://github.com/purenix-org/purenix) is a Nix backend for
[PureScript](https://www.purescript.org/).  PureNix allows you to write
PureScript code and transpile it to Nix code.

This post gives an overview of PureScript and PureNix. It then explains how to
get started using PureNix.

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
[alternative](https://github.com/purescript/documentation/blob/master/ecosystem/Alternate-backends.md)
code generators for other languages.  PureNix is an alternative
backend code generator for Nix.

## PureNix

PureNix works together with PureScript and Spago to compile PureScript code
into Nix code.  Here's a small example of PureScript code, and the Nix code it
gets compiled to:

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

PureNix compiles this PureScript code to the following Nix code[^modified]:

[^modified]: The formatting has been slightly cleaned up to make it a little
    easier to understand.

```nix
let
  greeting = "Hello, world!";

  identity = v: v;

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
in
  {inherit greeting Nothing Just fromMaybe;}
```

Except for the pattern matching in `fromMaybe`, this should look pretty
straight-forward.  You can see the PureScript string `greeting` becomes a
simple Nix string.  `identity` has become a Nix function.  The constructors for
`Maybe` (`Nothing` and `Just`) have become Nix functions[^tagged].  `fromMaybe` has
become a Nix function.  If you work through the `__patternX` continuations in
`fromMaybe`, you can see how they
[emulate the pattern matching](https://jonascarpay.com/posts/2021-11-08-nix-adts.html)
in the PureScript `fromMaybe` function.

[^tagged]: PureScript data constructors become tagged records in Nix.

Calling functions written in PureScript from Nix is quite fun.  It feels quite
similar to working in the Haskell or PureScript REPL.  Here is an example of
using the above functions from the Nix REPL:

```console
$ nix repl ./output/Main/default.nix
nix-repl> identity greeting
"Hello, world!"
nix-repl> fromMaybe "bye" (Just "hello")
"hello"
nix-repl> fromMaybe "bye" Nothing
"bye"
```

Now that you have a taste of PureNix, the next section explains how to get
started actually writing PureNix code.

## Getting Started

There is documentation in the PureNix repository that explains how to
put together a
[development environment](https://github.com/purenix-org/purenix/blob/main/docs/quick-start.md)
for PureNix.  It explains how to get to the point where you can actually start
writing PureScript and compiling it to Nix.

Once you have your environment setup, you'll likely be interested in what
libraries are available for PureNix.  The currently available libraries
are in the [purenix-org](https://github.com/purenix-org/) organization on
GitHub.  There are also two PureNix issues you may be interested in:

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

## Conclusion

PureNix gives you a way to write Haskell-like code and compile it to Nix.  This
is convenient for developers comfortable with Haskell-like languages, and
trying to write a non-trivial library in Nix.

*This post is the first post in a
[series about PureNix](./2022-01-03-purenix). The next post is about
[who would find PureNix easy to use](./2022-01-05-who-would-like-purenix).*

## Using PureNix Professionally

If your company is considering PureNix in order to tame a complicated Nix
codebase, [Jonas](https://jonascarpay.com/) and I currently have time available
for consulting or freelance.  Feel free to [get in touch](/about).  We are also
available for any other Nix/Haskell/PureScript-related work.
