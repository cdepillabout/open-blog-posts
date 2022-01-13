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

I was a mentor for [Summer of Nix 2021](https://summer.nixos.org/).  One of the
nice things about Summer of Nix is that the organizers arranged for various
people in the Nix community to give presentations about Nix-related topics.
One of the presentations was from [@adisbladis](https://github.com/adisbladis)
about [poetry2nix](https://github.com/nix-community/poetry2nix).

`poetry2nix` is a Nix library for building Python packages that use
[Poetry](https://python-poetry.org/) as a build system.  Poetry is a build tool
like Haskell's `stack`/`cabal`, Rust's `cargo`, etc.  Python packages using
Poetry must have a `pyproject.toml` file.  This file is similar to Haskell's
`.cabal` file, Rust's `Cargo.toml` file, etc.  Here's an
[example `pyproject.toml`](https://github.com/michaeloliverx/python-poetry-docker-example/blob/f7241bf6586e99c6c649eba36ca0efd935ea6316/pyproject.toml)
file. If you look at this file, you can see many things you'd expect in a
project configuration file, like dependencies and versions ranges.

Running Poetry produces a `poetry.lock` file, which locks all dependencies and
transitive dependencies to specific versions.  This lock file also contains
hashes for the source code of the dependency.  The `poetry.lock` file is in
[TOML format](https://en.wikipedia.org/wiki/TOML).  This `poetry.lock` file is
similar to Haskell's `stack.yaml.lock` file, Rust's `Cargo.lock` file, etc.

`poetry2nix` works by using Nix's `builtins.readFile` to read the raw
`pyproject.toml` and `poetry.lock` files from disk, and then uses
`builtins.fromTOML` to parse the TOML into plain Nix values.  `poetry2nix` then
uses this data to create Nix derivations for building all the dependencies, as well
as the package itself.  Having this all done in Nix and only using Nix builtins
means that `poetry2nix` does not need to use Import From Derivation (IFD).

I was quite surprised hearing an explanation of how `poetry2nix` works.  I had
thought that almost all languages needed to use IFD to take a native language
lock file and transform it into a Nix derivation.

## Import From Derivation (IFD) and Haskell

A quick explanation of how IFD works is that you first create a Nix derivation
that _outputs_ a `.nix` file.  If you build the derivation, the `.nix` that is created
will end up in the Nix store.  Within the same run of Nix, you then
_`import`_ this `.nix` file you just built.

If you'd like to see a more detailed introduction to IFD, checkout the following
two links:

- <https://blog.hercules-ci.com/2019/08/30/native-support-for-import-for-derivation/>
- <https://nixos.wiki/wiki/Import_From_Derivation>

There is a widely-used function in Nixpkgs that uses IFD to build a Haskell package:
[`haskellPackages.callCabal2nix`](https://github.com/NixOS/nixpkgs/blob/6a7bafcb0fd068716ca6659e59d197044c41a9a7/pkgs/development/haskell-modules/make-package-set.nix#L220).
Here's a rough explanation of how `callCabal2nix` works:

1.  A derivation is created that internally calls
    [`cabal2nix`](https://github.com/NixOS/cabal2nix).  `cabal2nix` reads in a
    input `.cabal` file and outputs a `.nix` file.  This `.nix` file is a
    derivation that can be built with the
    [default Haskell builder](https://github.com/NixOS/nixpkgs/blob/6a7bafcb0fd068716ca6659e59d197044c41a9a7/pkgs/development/haskell-modules/generic-builder.nix)
    in Nixpkgs.  `cabal2nix` uses the [`Cabal`](https://hackage.haskell.org/package/Cabal)
    Haskell library internally to parse the input `.cabal` file and get data
    out of it (in order to translate it to raw Nix code).
2.  Building the derivation from the previous step outputs a `.nix` file.
    This `.nix` file is _imported_ and passed to
    [`haskellPackages.callPackage`](https://github.com/NixOS/nixpkgs/blob/6a7bafcb0fd068716ca6659e59d197044c41a9a7/pkgs/development/haskell-modules/make-package-set.nix#L118).
3.  The derivation output from `haskellPackages.callPackage` in the previous
    step is built.  The output of this derivation is a normal Haskell library.

IFD is necessary in this process because of the first step.  In order to translate a
`.cabal` file into a Nix expression, the Haskell program `cabal2nix` needs to be
run.  `cabal2nix` uses the Haskell `Cabal` library internally.  It is necessary
to internally rely on the `Cabal` library because `.cabal` files are relatively
complicated.  Parsing a raw `.cabal` file directly with Nix would be
quite difficult.

After hearing that `poetry2nix` doesn't require IFD, I started thinking about
what would be necessary to write a `callCabal2nix` function that doesn't rely
on IFD.

## `callCabal2nix` Without IFD

In order to write a `callCabal2nix` function without relying on IFD, you would
first need to read in a `.cabal` file with the Nix `builtins.readFile` function,
then parse the raw `.cabal` file and pull out all the important information.
For building Haskell packages with Nix, the main information necessary from the
`.cabal` file is the list of direct dependencies.

The big difficulty in this process is parsing the `.cabal` file.  The `.cabal`
file format is not a format that Nix natively understands (like JSON or TOML).
The only widely-used library for parsing `.cabal` files is
[`Cabal`](https://hackage.haskell.org/package/Cabal), and this is only
available as a Haskell package.

If you wanted to parse a `.cabal` file with Nix, you would need to write a
`.cabal` parser in Nix itself.  This would be quite a challenge[^nix-parsec].

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
completely crazy idea.  After a little discussion, we came up with three
potential approaches for making a Haskell-like language that compiles to Nix:

1.  The alternative PureScript backend, as suggested above.

1.  Using [GHC's Core language](https://serokell.io/blog/haskell-to-core)
    as an intermediate representation, and translating that to Nix.

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

    Prior art here might be a project like [Clash](https://clash-lang.org/).

    The disadvantage of this approach is that it would be somewhat difficult to
    bootstrap the PureNix ecosystem.  We'd have to write our own standard library.
    If we went with an alternative PureScript backend, we could just rely on
    the PureScript standard library.

We decided to go with writing an alternative PureScript backend, hoping it
would be the quickest choice for actually writing a `callCabal2nix` function
that doesn't use IFD.

## Starting on PureNix

Writing an [alternative PureScript backend](https://github.com/purescript/documentation/blob/master/ecosystem/Alternate-backends.md)
is surprisingly easy[^altbackend].

[^altbackend]: TODO: assuming you are using the functional core and targeting a functional lang

TODO: explain how to write an alternative backend, and how long it took us

## Conclusion

*This post is the third post in a [series about PureNix](./2022-01-03-purenix).
The previous post is about
[who would find PureNix easy to use](./2022-01-05-who-would-like-purenix).*

## Using PureNix Professionally

If your company is considering PureNix in order to tame a complicated Nix
codebase, [Jonas](https://jonascarpay.com/) and I currently have time available
for consulting or freelance.  Feel free to [get in touch](/about).  We are also
available for any other Nix/Haskell/PureScript-related work.
