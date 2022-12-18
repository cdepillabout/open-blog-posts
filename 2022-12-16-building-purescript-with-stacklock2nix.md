------------------------------------------------------
title: Building the PureScript Compiler with stacklock2nix
summary: Use stacklock2nix to build the PureScript compiler.
tags: nixos, haskell, purescript
draft: false
------------------------------------------------------

*This is the first post in a
[series about `stacklock2nix`](./2022-12-15-stacklock2nix).*
The next post is about
[building Dhall with `stacklock2nix`](./2022-12-20-building-dhall-with-stacklock2nix).

<!-- The next post is about
[who would find PureNix easy to use](./2022-01-05-who-would-like-purenix). -->

This post uses [`stacklock2nix`](https://github.com/cdepillabout/stacklock2nix)
to build the PureScript compiler with Nix. PureScript is a non-trivial Haskell
project, but it is not _too_ complicated. It is good as an initial example of
using `stacklock2nix`.

The following commit adds a `flake.nix` to the PureScript repo that can be
used to build with Nix:

<https://github.com/cdepillabout/purescript/commit/b78324382f44e7d75ff5175e535df33a39f2e62e>

This `flake.nix` is based on the
[easy example](https://github.com/cdepillabout/stacklock2nix/tree/main/my-example-haskell-lib-easy)
of `stacklock2nix`.

This post explains how to use this `flake.nix`.

## What can you do with this `flake.nix`?

First, lets clone the above code locally with the following commands:

```console
$ git clone git@github.com:cdepillabout/purescript.git
$ cd purescript/
$ git checkout build-with-stacklock2nix
```

From here, you can use the `flake.nix` to build PureScript:

```console
$ nix build -L
```

After building, the compiler (called `purs`) should be available:

```console
$ ./result/bin/purs --version
0.15.7 [development build; commit: UNKNOWN]
```

You can also jump into a development shell that has tools like `cabal-install`
and HLS available:

```console
$ nix develop -L
```

From here, you can build PureScript with `cabal`:

```console
$ cabal build
...
$ cabal run exe:purs -- --version
0.15.7 [development build; commit: df5fcff1c396d520e8543d5d85ce1455e56e2696 DIRTY]
```

You can use other normal commands like `cabal test`, `cabal repl`, etc.

## Conclusion

PureScript is a large, non-trivial, single-package Haskell project.  Building
it using `stacklock2nix` is relatively straight-forward.  Few Nix overrides are
necessary for getting everything to build.  `stacklock2nix` makes it easy to
build a Haskell project with Nix, given that you already have a `stack.yaml`
and `stack.yaml.lock` file.
