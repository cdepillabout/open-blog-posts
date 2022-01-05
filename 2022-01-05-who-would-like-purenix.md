------------------------------------------------------
title: Who Would Find PureNix Easy to Use?
summary: What sort of developers would find PureNix easy to get started with?
tags: haskell, nixos, purescript
draft: true
------------------------------------------------------

*This post is the second post in a
[series about PureNix](./2022-01-03-purenix).  The previous post was about
[getting started with PureNix](./2022-01-04-getting-started-with-purenix).*

PureNix will be easy to get started with for developers that have a good
understanding of both Haskell-like languages and Nix.  Developers without this
background will have some trouble using PureNix.

This blog post gives some suggestions about how developers with different
knowledge sets may want to approach PureNix.

## Developers that Know Haskell, PureScript, and Nix

If you have experience with Haskell, PureScript, and Nix, PureNix will be easy
to get started with.  PureNix should feel quite natural.

There are two things you may want to keep in mind:

-   You may have to write some FFI for Nix builtins.

    Here's an
    [example PureScript module](https://github.com/cdepillabout/cabal2nixWithoutIFD/blob/484515bdec2ccf9dfc02b9a442b801bc2d17b9cc/purescript-parser-combinator/src/NixBuiltins.purs)
    that defines some Nix builtins, as well as the underlying
    [FFI code](https://github.com/cdepillabout/cabal2nixWithoutIFD/blob/484515bdec2ccf9dfc02b9a442b801bc2d17b9cc/purescript-parser-combinator/src/NixBuiltins.nix).

-   Nix is a pure language, so there is nothing like Haskell's `IO` type or
    PureScript's `Effect` type in PureNix.  The fact that you can have
    all your functions be pure in PureScript works out really nicely with
    PureNix and Nix.

## Developers that Know PureScript and Nix (but not Haskell)

If you have experience with PureScript and Nix (but no experience with
Haskell), PureNix should still be pretty easy.  Unlike the main JavaScript
backend for PureScript, PureNix is lazy.  This will feel a little weird at
first, but since you already know Nix, it shouldn't be too difficult.

There are also a few type-classes in PureScript that aren't necessary in
PureNix, like:

- [`Lazy`](https://pursuit.purescript.org/packages/purescript-control/5.0.0/docs/Control.Lazy#t:Lazy)
- [`MonadRec`](https://pursuit.purescript.org/packages/purescript-tailrec/5.0.1/docs/Control.Monad.Rec.Class)

## Developers that Know Haskell and Nix (but not PureScript)

If you have experience with Haskell and Nix (but not PureScript), it
might take you a little bit of time to get used to the PureScript
standard library. But other than that, you shouldn't have many problems.

In general, PureScript is split up into a large number of small packages and
modules, much more so than Haskell.  For example, `Maybe` and `Either` live
in completely separate packages. They are not included the `Prelude`:

-   [purescript-maybe](https://github.com/purenix-org/purescript-maybe)
-   [purescript-either](https://github.com/purenix-org/purescript-either)

There are many common data-types and functions that are in their
own packages.  But most data-types and functions are named similarly to their
counterparts in Haskell, so you shouldn't have much trouble in practice.
You'll likely find the following resources very useful:

-   [Pursuit](https://pursuit.purescript.org/).  A combination of a search engine
    like Hoogle, and API documentation like Hackage (but for PureScript instead
    of Haskell).[^pursuit]
-   [Differences from Haskell](https://github.com/purescript/documentation/blob/master/language/Differences-from-Haskell.md).
    Official documentation on the differences between PureScript and Haskell.
-   [Language Documentation](https://github.com/purescript/documentation/tree/master/language).
    Official documentation on the PureScript language.  You'll likely want to
    focus on the document about Records, since they are one of the main
    differences between PureScript and Haskell.

[^pursuit]: Pursuit doesn't know anything about PureNix, but the API of the
    PureNix libraries that have been ported so far is almost the same as the
    PureScript version of the libraries, so searching for functions on Pursuit is
    still helpful.  In the future it would be nice if there was a Pursuit
    specifically for PureNix.

## Developers that Know Haskell or PureScript (but not Nix)

If you have experience with Haskell or PureScript (but not Nix), you may
have some trouble getting started.  It may be a little early to attempt
to use PureNix.  You would likely benefit from first reading the
[Nix Manual](https://nixos.org/manual/nix/stable/).

Nix feels like a simple lambda calculus (along with some built-in types and
functions), so if you're already familiar with a Haskell-like language,
it shouldn't take you that long to learn Nix.  Although, learning
[Nixpkgs](https://github.com/NixOS/nixpkgs) is much more time-consuming.

## Developers that Know Nix (but not Haskell or PureScript)

If you have experience with Nix (but not Haskell or PureScript), you will
likely have trouble getting started with PureNix.  I would recommend
first trying to learn Haskell or PureScript.  If your goal is to use PureNix,
then it doesn't really matter if you first learn Haskell or PureScript, since
they are so similar.  Haskell seems to have more beginner-oriented resources
than PureScript.

Keep in mind that like Nix, Haskell and PureScript have a steep learning curve.
Becoming proficient in Haskell or PureScript can be quite time-consuming.

Here are a few resources I've heard people recommend:

-   [Haskell from First Principles](https://haskellbook.com/).  This book is
    long, but quite thorough.
-   [Programming in Haskell](https://www.cs.nott.ac.uk/~pszgmh/pih.html).  Lots of
    people seem to like this book.
-   [PureScript by Example](https://book.purescript.org/).  The canonical
    PureScript learning resource.

You would probably benefit from doing a Google search on the most recommended
learning materials.

## Conclusion

PureNix should be easy to get started with if you know Haskell/PureScript and
Nix.  You may have a harder time if you're new to either Haskell-like
languages, or Nix.

*This post is the second post in a
[series about PureNix](./2022-01-03-purenix).  The previous post was about
[getting started with PureNix](./2022-01-04-getting-started-with-purenix).*

## Using PureNix Professionally

If your company is considering PureNix in order to tame a complicated Nix
codebase, [Jonas](https://jonascarpay.com/) and I currently have time available
consulting or freelance.  Feel free to [get in touch](/about).  We are also
available for any other Nix/Haskell/PureScript-related work.
