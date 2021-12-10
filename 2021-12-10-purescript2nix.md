------------------------------------------------------
title: purescript2nix
summary: Easily build a PureScript project with Nix
tags: nixos, purescript
draft: false
------------------------------------------------------

*This post is part of a series called
[The Road to `purescript2nix`](./2021-12-10-road-to-purescript2nix).*

[`purescript2nix`](https://github.com/cdepillabout/purescript2nix)
is a Nix function that allows you to easily build
a PureScript project with Nix.  It uses the info in your `spago.dhall`
and `packages.dhall` files as a single source of truth in order to
construct a Nix build plan.

This post explains how to use `purescript2nix`, shows some of the
implementation, and explains why I started working on this project.

## Using `purescript2nix`

Given a PureScript project in the directory `./example-purescript-package`,
you can build it with `purescript2nix` with a call like the following[^1]:

```nix
purescript2nix {
  pname = "example-purescript-package";
  src = ./example-purescript-package;
}
```

This produces a derivation that will build your source code.

The `purescript2nix` repository contains an
[example package](https://github.com/cdepillabout/purescript2nix/blob/d16ed5b38a26ea72d114cc0e3df0db5bd20e902b/nix/overlay.nix#L8-L17)
that you may want to play around with.

## Implementation

Under the covers, `purescript2nix` will roughly do the following things:

1. Read the `spago.dhall` and `packages.dhall` files into Nix.

    This is done with the Nix function
    [`dhallDirectoryToNix`](./2021-12-10-dhallDirectoryToNix).

2. Compute all transitive dependencies for the package
    you are trying to build, given an input package set.

    This is mainly done with the Nix function
    `builtins.genericClosure`[^2].

3. Download the PureScript source code for all dependencies.

    This is relatively straight-forward with Nix.

4. Actually compile the PureScript code.

    This is done by directly calling `purs`.

The output of `purescript2nix` is a derivation that contains all the compiled
PureScript code.

## Why write `purescript2nix`?

I was a mentor for [Summer of Nix 2021](https://summer.nixos.org/).
One of the members on my team, David Hauer ([@DavHau](https://github.com/DavHau)),
started the project [dream2nix](https://github.com/nix-community/dream2nix).
dream2nix is meant to be a generic framework to wrap up all the _lang2nix_
tools (like [`cabal2nix`](https://github.com/NixOS/cabal2nix),
[`crate2nix`](https://github.com/kolloch/crate2nix),
`yarn2nix`, [`poetry2nix`](https://github.com/nix-community/poetry2nix), etc) and
provide a unified interface.

I wanted to contribute a backend to dream2nix for PureScript, but it seemed
like there weren't any easy approaches to building a PureScript project with
Nix.  Most current approaches require a manual steps for dumping the contents
of the `spago.dhall` and `packages.dhall` files, as well getting Nix-compatible
hashes for all PureScript dependencies.

I wanted to eliminate these manual steps, so I started thinking about
what would be required for an easy-to-use `purescript2nix` function.
The two big things necessary for this are:

-   the ability to
    [read Dhall files in as Nix](./2021-12-10-dhallDirectoryToNix)
    (even when the Dhall files have remote imports)
-   [Nix-compatible hashes](./2021-12-10-purescript-package-set-with-hashes)
    in the PureScript package sets

Implementing these two things took a surprising amount of time. But with
these in place, putting together `purescript2nix` was relatively easy.

Although, after this work, I ran out of steam and didn't end up creating a
PureScript backend for `dream2nix`.  I have
[opened an issue](https://github.com/cdepillabout/purescript2nix/issues/5)
about it in case anyone wants to work on this.

## Caveats

Currently, the big caveat with `purescript2nix` is that it requires
using a PureScript package set with Nix-compatible hashes.
See [this issue](https://github.com/cdepillabout/purescript2nix/issues/4)
for more information.

There is also [an issue](https://github.com/cdepillabout/purescript2nix/issues/1)
with using `purescript2nix` when building from a Nix Flake.

I would welcome help on either of these issues (or anything else on the
`purescript2nix` issue tracker).

## Conclusion

If you're looking for an easy way to build a PureScript package with
Nix, `purescript2nix` might be a good choice.

If you're looking to adopt `purescript2nix` into your build system
professionally, I am open to freelancing or consulting arrangements.
I am of course open to any other freelance or consulting projects involving Nix.
Feel free to [get in touch](https://functor.tokyo/about) for more info.

## Footnotes

[^1]: This requires access to the `purescript2nix` function.  You can get access
    to this function by adding the
    [`purescript2nix` repo](https://github.com/cdepillabout/purescript2nix)
    as a Flake input, or just directly importing the repo.  See the
    [README.md](https://github.com/cdepillabout/purescript2nix#readme) for more
    info.  [Open an issue](https://github.com/cdepillabout/purescript2nix/issues)
    if you need more help.

[^2]: I learned about this from Justin Woo.  He has a post about this called
    [Working with PureScript package sets with just Nix](https://qiita.com/kimagure/items/25ca3ddcc8e0b636884e).
    I put together my own example of `builtins.genericClosure` in
    [this issue](https://github.com/NixOS/nix/issues/552#issuecomment-971212372).
