------------------------------------------------------
title: Building Spago with stacklock2nix
summary: Use stacklock2nix to build Spago while statically linking
tags: nixos, haskell
draft: false
------------------------------------------------------

*This is the third and final post in a
[series about `stacklock2nix`](./2022-12-15-stacklock2nix).
The previous post is about
[building Dhall with `stacklock2nix`](./2022-12-20-building-dhall-with-stacklock2nix).*

This post uses [`stacklock2nix`](https://github.com/cdepillabout/stacklock2nix)
to build [Spago](https://github.com/purescript/spago) with Nix. Spago is a
popular build tool for PureScript.
example for `stacklock2nix`, since it is comprised of quite a few different
Haskell packages.

The following commit adds some Nix code to the Dhall repo that can be used to
build with Nix[^1]:

<https://github.com/cdepillabout/dhall-haskell/commit/e1a7c31d68ad2da9c748dddcca01668da0ca53d6>

The most interesting file in this commit is `nix/overlay.nix`.  This is the
file I recommend taking a look at to get a feel for using `stacklock2nix`.
This newly-added Nix code is based on the
[advanced example](https://github.com/cdepillabout/stacklock2nix/tree/main/my-example-haskell-lib-advanced)
of `stacklock2nix`.  This post explains how to use this newly-added Nix code.

## What can you do with this?

First, clone the above repo locally with the following commands:

```console
$ git clone git@github.com:cdepillabout/dhall-haskell.git
$ cd dhall-haskell/
$ git checkout build-with-stacklock2nix
$ git submodule update --init
```

From here, you can use the `flake.nix` file to build the Dhall tools:

```console
$ nix build -L
```

After building, all the Dhall tools should be available:

```console
$ ls result/bin/
csv-to-dhall
dhall
dhall-docs
dhall-lsp-server
dhall-to-bash
dhall-to-csv
dhall-to-json
dhall-to-nix
...
```

You can also use the `default.nix` file with `nix-build`:

```console
$ nix-build
```

You can use `nix develop` to jump into a development shell that has tools like
`cabal-install` and HLS available:

```console
$ nix develop -L
```

Or you could do the same thing with `nix-shell`:

```console
$ nix-shell
```

From here, you can build all the Dhall tools with `cabal`:

```console
$ cabal build all
...
$ cabal run dhall -- --version
1.41.2
```

You can also use other normal commands like `cabal test`, `cabal repl`, etc.

The repo is also setup to be able to build with `stack`[^2]:

```console
$ stack --nix build
```

And run tests:

```console
$ stack --nix test
```

## Conclusion

Dhall is a non-trivial, multi-package Haskell project.  Building
it with `stacklock2nix` is relatively straight-forward.  A few Nix overrides are
necessary for getting everything to build.  As long as you have a `stack.yaml`
and `stack.yaml.lock` file, `stacklock2nix` makes it easy to build a Haskell
project with Nix.  Even a large, multi-package Haskell project.

The Haskell dependency versions are controlled by the `stack.yaml` file.
Bumping the Stackage resolver in `stack.yaml` is all that is needed to build
with newer Haskell dependencies.  This is much easier than manually keeping
dependencies in sync between Haskell and Nix.

## Footnotes

[^1]: Dhall actually already has a nice infrastructure for building with Nix.
    In practice, this existing infrastructure should be used instead of
    `stacklock2nix`.  The commit from this blog post is  purely to show how
    `stacklock2nix` could be used on a non-trivial Haskell project.  (There is
    also a parent commit that deletes all the existing Nix infrastructure in
    the Dhall repo, just to make everything a little easier to understand.)

[^2]: Note that `stack` internally runs `nix-shell`, so you can run `stack`
    outside of `nix-shell`.  That is to say, you don't have to be inside
    `nix-shell` before you run any of the `stack` commands.
