------------------------------------------------------
title: Why PureNix?
summary: What lead to us starting to write PureNix?
tags: haskell, nixos, purescript
draft: false
------------------------------------------------------

*This post is the third post in a [series about PureNix](./2022-01-03-purenix).
The previous post is about
[who would find PureNix easy to use](./2022-01-05-who-would-like-purenix).*

[PureNix](https://github.com/purenix-org/purenix) started out half as a joke.
This post explains why we started working on PureNix, and how it moved from a
joke to something we are excited about.

## The Idea

This all started during [@adisbladis](https://github.com/adisbladis)'s
 [Summer of Nix 2021](https://summer.nixos.org/) presentation about
 [poetry2nix](https://github.com/nix-community/poetry2nix).

Briefly, Poetry is a Python build system, akin to Haskell's Cabal or Rust's
Cargo.  Poetry is fairly unremarkable; you define your project in a
`pyproject.toml` file (like Cabal's `.cabal` files, or Cargo's `Cargo.toml`),
and when you first build your project it freezes all (transitive) dependencies
in a `poetry.lock` file (like Cargo's `Cargo.lock`).

It seems that that would make `poetry2nix` to Poetry what `cabal2nix` is to
Cabal, or `cargo2nix` to Cargo, but `poetry2nix` is special in an important
way.  The way `poetry2nix` works is it uses Nix's `builtins.readFile` to read
the raw `pyproject.toml` and `poetry.lock` files from disk, and then uses
`builtins.fromTOML` to parse the TOML into plain Nix values.  It then uses this
data to create Nix derivations for building all the dependencies, as well as
the package itself.

What's special about this is not so much what it _does_, but what it _doesn't_
do.  Because `poetry2nix` is implemented entirely in Nix and only uses Nix
builtins, it never uses Import From Derivation (IFD).  Up until this
presentation, I thought that all translation layers like `cabal2nix` used IFD
to take a native language lock file and transform it into a Nix derivation. I
hadn't even considered that you could get away with not doing that.

## Import From Derivation (IFD) and Haskell

A quick explanation of IFD:

1.  You create a Nix derivation that _outputs_ a `.nix` file.
2.  You build this derivation, and the `.nix` file that is created end up in
    the Nix store (since it is a build output).
3.  Within the same run of Nix, you _`import`_ the `.nix` file from its path in
    the Nix store.

Checkout the following two links for a more detailed introduction of IFD:

- <https://blog.hercules-ci.com/2019/08/30/native-support-for-import-for-derivation/>
- <https://nixos.wiki/wiki/Import_From_Derivation>

There is a widely-used function in Nixpkgs that uses IFD to build a Haskell package:
[`haskellPackages.callCabal2nix`](https://github.com/NixOS/nixpkgs/blob/6a7bafcb0fd068716ca6659e59d197044c41a9a7/pkgs/development/haskell-modules/make-package-set.nix#L220).
Roughly, `callCabal2nix` works by running a Haskell program to parse an input
`.cabal` file and pull out all necessary information for building the Haskell
package with Nix.  IFD is necessary in this process because the only library
for parsing a `.cabal` file is written in Haskell.  Parsing a raw `.cabal` file
directly within Nix would be quite difficult.

After hearing that `poetry2nix` doesn't require IFD, I started thinking about
what would be necessary to write a `callCabal2nix` function that doesn't rely
on IFD.

## `callCabal2nix` Without IFD

In order to write a `callCabal2nix` function without relying on IFD, you would
first need to read in a `.cabal` file with the Nix `builtins.readFile` function,
then parse the raw `.cabal` file and pull out all the important information
from within Nix.

The big difficulty in this process is parsing the `.cabal` file.  If you wanted
to parse a `.cabal` file with Nix, you would need to write a `.cabal` parser in
Nix itself.  This would be quite a challenge[^nix-parsec].

[^nix-parsec]: There is a Nix library called
    [`nix-parsec`](https://github.com/nprindle/nix-parsec) that implements a
    [`parsec`](https://hackage.haskell.org/package/parsec)-like library in raw
    Nix.  This could potentially be used as a base, but it would still be
    non-trivial to write a full `.cabal` parser in raw Nix.

I was thinking that if I was going to write a `callCabal2nix` function that
doesn't use IFD, I would want to write it in a Haskell-like language that
provides features like type-checking, algebraic data types, pattern-matching,
type classes, etc.  This language would need to compiled to Nix code, so
that Nix can execute it.

My first thought was to write a Nix backend for PureScript.  Users would be able
to write PureScript code and compile it to Nix.  This seemed like somewhat of a
silly idea, so I decided to share it with my friend, [Jonas Carpay](https://jonascarpay.com/).

## Enter Jonas

Jonas is a Haskeller, he's interested in compilers and programming languages,
and he's a heavy Nix user.  I thought that if there is anyone I could convince
to work on this with me, it would be Jonas.

After telling Jonas about this, he surprisingly didn't think this was a
crazy idea.  After a little discussion, we came up with three
potential approaches for making a Haskell-like language that compiles to Nix:

1.  The alternative PureScript backend, as suggested above.

1.  Use [GHC's Core language](https://serokell.io/blog/haskell-to-core)
    as an intermediate representation, and translate that to Nix.

    This approach would mean that the user would directly write a program in
    Haskell.  Our compiler would use GHC to compile the Haskell program to GHC
    Core.  Our compiler would then transpile this GHC Core to Nix.

    The advantage of this approach is that the user would be able to use all of
    Haskell's features, even things like GADTs, type families, etc.

    The disadvantage of this approach is that neither of us had ever really worked
    with GHC Core before.  We weren't sure how hard it would be to translate
    Core into Nix, or what the consequences would be for the
    [GHC Boot Libraries](https://gitlab.haskell.org/ghc/ghc/-/wikis/commentary/libraries/version-history)
    that are shipped with the compiler.  We weren't sure of all the primitives
    exposed by GHC, or how these would translate to Nix.

    I know there are compilers like [GHCJS](https://github.com/ghcjs/ghcjs) and
    [Eta](https://eta-lang.org/) that attempt to hook into some step in GHC's
    compilation pipeline and output to a separate language (JavaScript in the case
    of GHCJS, and Java in the case of Eta).  But my image of these projects is
    that they are quite complicated.

1.  Write a DSL in Haskell that outputs Nix code when run.

    Prior art here might be a project like [hnix](https://github.com/haskell-nix/hnix).

    The disadvantage of this approach is that it would be somewhat difficult to
    bootstrap our new ecosystem.  We would have to write our own standard library.
    If we went with an alternative PureScript backend, we could just rely on
    the PureScript standard library.

We decided to go with writing an alternative PureScript backend, hoping it
would be the quickest choice for actually writing a `callCabal2nix` function
that doesn't use IFD.

## Starting on PureNix

Writing an [alternative PureScript backend](https://github.com/purescript/documentation/blob/master/ecosystem/Alternate-backends.md)
is surprisingly easy[^altbackend].  This section gives a short introduction to
what is necessary for writing an alternative backend.

[^altbackend]: This is assuming you are using PureScript's
    [functional Core language](https://hackage.haskell.org/package/purescript-0.13.8/docs/Language-PureScript-CoreFn-Module.html),
    and you are targeting a dynamically-typed functional language (like Nix).
    If your target language is statically-typed, or an imperative language,
    my guess is that writing an alternative PureScript backend would be more
    difficult.  Although there are quite a few alternative PureScript backends for
    [statically-typed, imperative languages](https://github.com/purescript/documentation/blob/master/ecosystem/Alternate-backends.md).

The PureScript compiler defines a
[functional](https://hackage.haskell.org/package/purescript-0.13.8/docs/Language-PureScript-CoreFn.html)
Core language[^multicore].  An alternative PureScript backend is responsible for taking a
functional Core
[`Module`](https://hackage.haskell.org/package/purescript-0.13.8/docs/Language-PureScript-CoreFn-Module.html)
and converting it into a module in your target language.  PureNix has three
main parts that do this conversion into Nix code:

[^multicore]: The PureScript compiler actually defines two separate Core languages: a
    [functional](https://hackage.haskell.org/package/purescript-0.13.8/docs/Language-PureScript-CoreFn.html)
    Core language and an
    [imperative](https://hackage.haskell.org/package/purescript-0.13.8/docs/Language-PureScript-CoreImp.html)
    Core language.  There is a flag you can pass to the PureScript compiler to
    have it output the functional Core language instead of JavaScript code
    when compiling.  Spago does this for you internally when you specify
    a [`backend`](https://github.com/purescript/spago#use-alternate-backends-to-compile-to-go-c-kotlin-etc).
    See the [Getting Started Guide](https://github.com/purenix-org/purenix/blob/main/docs/quick-start.md)
    for PureNix for a little more information about setting a `backend`.

-   A definition of an [AST for Nix](https://github.com/purenix-org/purenix/blob/b6bf56a20b26b9744b207bed75268c09dd611b79/src/PureNix/Expr.hs)
-   A [function](https://github.com/purenix-org/purenix/blob/b6bf56a20b26b9744b207bed75268c09dd611b79/src/PureNix/Convert.hs#L46-L51)
    for converting a PureScript `Module` into our Nix AST
-   A [function](https://github.com/purenix-org/purenix/blob/b6bf56a20b26b9744b207bed75268c09dd611b79/src/PureNix/Print.hs)
    for taking our Nix AST and converting to raw Nix code

This is all there is to it.  PureScript's functional Core language translates
quite nicely to Nix, so we didn't have too much trouble here.  The only
difficulty is how to encode PureScript's data types and pattern-matching to
Nix.  Jonas came up with a
[good solution](https://jonascarpay.com/posts/2021-11-08-nix-adts.html) for this.

Writing PureNix only took about a month.  The end result was much better than
either of us had anticipated.  PureNix ends up working out really well in
practice.  The Nix code it outputs is very similar to what you'd write by
hand[^typeclasses].

[^typeclasses]: Well, other than type classes and pattern matching.  Both of these can
    be a little verbose and hard to decipher in the output Nix code.

With the PureNix compiler mostly finished, the next step was to port some
PureScript standard libraries over to be used with PureNix.

## Porting PureScript Libraries

Unlike a language like Haskell or Python, PureScript doesn't have a big
"standard library"[^haskell-stdlib].  However, there is a set of about 60 PureScript libraries
maintained under the [`purescript`](https://github.com/purescript) organization
on GitHub (all the repositories with the `purescript-` prefix).  This set of
libraries is often thought of as the "standard library" for PureScript.  When
writing an alternative backend, the first step is porting some of these
libraries to your new backend.

[^haskell-stdlib]: In Haskell, sometimes people think of
    [base](https://hackage.haskell.org/package/base) as the
    standard library.  Sometimes people think of the full set of
    [GHC Boot Libraries](https://gitlab.haskell.org/ghc/ghc/-/wikis/commentary/libraries/version-history)
    as the standard library, since these are all shipped with the compiler.

After getting the PureNix compiler mostly working, we started on the process of
porting some of the above libraries to PureNix.  This process mostly consists
of forking the repository and rewriting all the JavaScript FFI files to Nix
(the PureScript source files can mostly be used as-is).  This is somewhat
annoying and time-consuming, but it is not particularly difficult.  The
libraries that have been ported work well when called from Nix.  It almost
feels like magic being able to call functions written in PureScript from Nix.

We ended up porting about 25 libraries so far.  We worked on this
on and off, and it ended up taking about 2 months.  See
[this issue](https://github.com/purenix-org/purenix/issues/37) for the status
of the porting process for the remaining libraries.  Feel free to jump in and help!

With some libraries ported, the next step was to actually start writing a
version of `callCabal2Nix` that doesn't need IFD!

## No IFD

With a portion of the PureScript standard library available, writing a
[proof-of-concept `.cabal` parser](https://github.com/cdepillabout/cabal2nixWithoutIFD)
was straight-forward.  This project currently only parses a small
subset of the full `.cabal` file syntax, but this approach should be extendable
to work with a full `.cabal` file.  This project accomplishes the goal of
parsing a `.cabal` file within Nix, without using IFD.  This whole process ends
up being quite similar to `poetry2nix`.

I plan to write a blog post about `cabal2nixWithoutIFD` in the future, but if
you're interested, checkout the
[`README.md`](https://github.com/cdepillabout/cabal2nixWithoutIFD) in the repo.

## Conclusion

While PureNix started out half as a joke, it ended up working out much better
than anticipated.  This is mostly due to the similarity between PureScript's
functional Core language and Nix.  PureScript's standard libraries also work
well when compiled to Nix.  It is quite nice to effectively be writing Nix,
but using things like type-checking, algebraic data types, pattern-matching,
and type classes.  Relying on PureScript's standard libraries for writing Nix
code is quite convenient, given that PureScript's standard libraries provide
quite a lot of features.

The whole PureNix project ended up working out really well, and we hope that
other people will be able to find a use for PureNix as well.

*This post is the third post in a [series about PureNix](./2022-01-03-purenix).
The previous post is about
[who would find PureNix easy to use](./2022-01-05-who-would-like-purenix).*

## Using PureNix Professionally

If your company is considering PureNix in order to tame a complicated Nix
codebase, [Jonas](https://jonascarpay.com/) and I currently have time available
for consulting or freelance.  Feel free to [get in touch](/about).  We are also
available for any other Nix/Haskell/PureScript-related work.
