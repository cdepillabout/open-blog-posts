------------------------------------------------------
title: Building Pandoc with stacklock2nix
summary: Use stacklock2nix to build a fully statically-linked Pandoc
tags: nixos, haskell
draft: false
------------------------------------------------------

*This is the third and final post in a
[series about `stacklock2nix`](./2022-12-15-stacklock2nix).
The previous post is about
[building Dhall with `stacklock2nix`](./2022-12-20-building-dhall-with-stacklock2nix).*

This post uses [`stacklock2nix`](https://github.com/cdepillabout/stacklock2nix)
to build [Pandoc](https://github.com/jgm/pandoc) with Nix. Pandoc is a tool for
converting from one markup format to another.  It is one of the most widely-used
tools written in Haskell. This post explains how to use `stacklock2nix` to
build Pandoc, but with a twist that Pandoc will be fully statically-linked.
This means that you can take the Pandoc binary built with Nix and use it
on any Linux distribution.

The following commit adds a `flake.nix` to the Pandoc repo that can be used to
build with Nix and statically-link:

<!-- TODO: -->
<https://github.com/cdepillabout/pandoc/commit/>

This `flake.nix` is based on the
[easy example](https://github.com/cdepillabout/stacklock2nix/tree/main/my-example-haskell-lib-easy)
of `stacklock2nix`. This post explains how to use this `flake.nix`.

## What can you do with this `flake.nix`?

First, clone the above repo locally with the following commands:

```console
$ git clone git@github.com:cdepillabout/pandoc.git
$ cd pandoc/
$ git checkout build-with-stacklock2nix
```

This `flake.nix` is very similar to the one added for
[building PureScript](./2022-12-16-building-purescript-with-stacklock2nix).
You can run all the same commands, like `nix build`, or `nix develop` and then
`cabal build all`.

The big difference is that this `flake.nix` for Pandoc is setup to statically-link
the binaries it produces.  So if you build the binary, you can see that it has
been statically-linked:

```console
$ nix build
...
$ file result/bin/pandoc
result/bin/pandoc: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```

You can see that it says "statically linked".


```console
$ docker run -it -v `pwd`/result:/pandoc ubuntu /pandoc/bin/pandoc --version
pandoc 3.0
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
