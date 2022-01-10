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
`builtins.fromTOML` parse the TOML into plain Nix values.  `poetry2nix` then
uses this data to create Nix derivations for building all the dependencies, as well
as the package itself.  Having this all done in Nix and only using Nix builtins
means that `poetry2nix` does not need to use Import From Derivation (IFD).

I was quite surprised hearing an explanation of how `poetry2nix` works.  I had
thought that almost all languages needed to use IFD to take a native language
lock file and transform it into a Nix derivation.

## Import From Derivation (IFD) and Haskell

A quick explanation of how IFD works is that you first create a Nix derivation
that _outputs_ a `.nix` file.  If you build the derivation, then your created
`.nix` file will end up in the Nix store.  Within the same run of Nix, you then
_`import`_ this `.nix` file you just built.

If you'd like to see a more detailed introduction to IFD, checkout the following
two links:

- <https://blog.hercules-ci.com/2019/08/30/native-support-for-import-for-derivation/>
- <https://nixos.wiki/wiki/Import_From_Derivation>

There is a widely-used function in Nixpkgs that uses IFD for a Haskell package:
[`haskellPackages.callCabal2nix`](https://github.com/NixOS/nixpkgs/blob/6a7bafcb0fd068716ca6659e59d197044c41a9a7/pkgs/development/haskell-modules/make-package-set.nix#L220).
Here's a rough explanation of how `callCabal2nix` works:

1.  A derivation is created that internally calls
    [`cabal2nix`](https://github.com/NixOS/cabal2nix).  `cabal2nix` reads in a
    `.cabal` file and outputs a `.nix` file.  This `.nix` file is a derivation
    that can be built with the
    [default Haskell builder](https://github.com/NixOS/nixpkgs/blob/6a7bafcb0fd068716ca6659e59d197044c41a9a7/pkgs/development/haskell-modules/generic-builder.nix)
    in Nixpkgs.  `cabal2nix` uses the [`Cabal`](https://hackage.haskell.org/package/Cabal)
    Haskell library internally to parse the input `.cabal` file and get data out of it.
2.  Building the derivation from the previous step outputs a `.nix` file.
    This `.nix` file is _imported_ and passed to
    [`haskellPackages.callPackage`](https://github.com/NixOS/nixpkgs/blob/6a7bafcb0fd068716ca6659e59d197044c41a9a7/pkgs/development/haskell-modules/make-package-set.nix#L118).




## Conclusion

*This post is the third post in a [series about PureNix](./2022-01-03-purenix).
The previous post is about
[who would find PureNix easy to use](./2022-01-05-who-would-like-purenix).*

## Using PureNix Professionally

If your company is considering PureNix in order to tame a complicated Nix
codebase, [Jonas](https://jonascarpay.com/) and I currently have time available
for consulting or freelance.  Feel free to [get in touch](/about).  We are also
available for any other Nix/Haskell/PureScript-related work.
