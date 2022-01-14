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
crazy idea.  After a little discussion, we came up with three
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
    bootstrap the PureNix ecosystem.  We would have to write our own standard library.
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

The PureScript compiler defines two separate Core languages: a
[functional](https://hackage.haskell.org/package/purescript-0.13.8/docs/Language-PureScript-CoreFn.html)
Core language and an
[imperative](https://hackage.haskell.org/package/purescript-0.13.8/docs/Language-PureScript-CoreImp.html)
Core language.  There is a flag you can pass to the PureScript compiler to
have it output the functional Core language instead of JavaScript code
when compiling.  Spago does this for you internally when you specify
a [`backend`](https://github.com/purescript/spago#use-alternate-backends-to-compile-to-go-c-kotlin-etc).
See the [Getting Started Guide](https://github.com/purenix-org/purenix/blob/main/docs/quick-start.md)
for PureNix for a little more information about setting a `backend`.

An alternative PureScript backend is responsible for taking a PureScript
[`Module`](https://hackage.haskell.org/package/purescript-0.13.8/docs/Language-PureScript-CoreFn-Module.html)
and converting it into a module in your target language.  PureNix has three
main parts that do this conversion into Nix code:

-   A definition of an [AST for Nix](https://github.com/purenix-org/purenix/blob/b6bf56a20b26b9744b207bed75268c09dd611b79/src/PureNix/Expr.hs)
-   A [function](https://github.com/purenix-org/purenix/blob/b6bf56a20b26b9744b207bed75268c09dd611b79/src/PureNix/Convert.hs#L46-L51)
    for converting a PureScript `Module` into our Nix AST
-   A [function](https://github.com/purenix-org/purenix/blob/b6bf56a20b26b9744b207bed75268c09dd611b79/src/PureNix/Print.hs) for taking our Nix AST and converting to raw Nix code

This is all there is to it.  PureScript's functional Core langauge translates
quite nicely to Nix, so we didn't have too much trouble here.  The only
difficulty is how to encode PureScript's data types and pattern-matching to
Nix.  Jonas came up with a
[good solution](https://jonascarpay.com/posts/2021-11-08-nix-adts.html) for this.

Writing PureNix only took about a month.  The end result was much better than
either of us had anticipated.  PureNix ends up working out really well in
practice.  The Nix code it outputs is very similar to what you'd write by
hand[^typeclasses].

[^typeclasses]: Other than typeclasses and pattern matching.  Both of these can
    be a little verbose and hard to decipher in the output Nix code.

With PureNix mostly finished, the next step was to port some PureScript
standard libraries over to be used with PureNix.

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
    as the standard library.

After getting the PureNix compiler mostly working, we started on the process of
porting some of the above libraries to PureNix.  This process mostly consists
of forking the repository and rewriting all the JavaScript FFI files to Nix.
This is somewhat annoying and time-consuming, but it is not particularly
difficult.  We ended up porting about 25 libraries so far.  We worked on this
on and off.  This ended up taking about 2 months.  See
[this issue](https://github.com/purenix-org/purenix/issues/37) for the status
of the porting process for the remaining.  Feel free to jump in and help!

With 

## Conclusion

*This post is the third post in a [series about PureNix](./2022-01-03-purenix).
The previous post is about
[who would find PureNix easy to use](./2022-01-05-who-would-like-purenix).*

## Using PureNix Professionally

If your company is considering PureNix in order to tame a complicated Nix
codebase, [Jonas](https://jonascarpay.com/) and I currently have time available
for consulting or freelance.  Feel free to [get in touch](/about).  We are also
available for any other Nix/Haskell/PureScript-related work.
