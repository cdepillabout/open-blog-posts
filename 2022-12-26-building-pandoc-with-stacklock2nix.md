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

<https://github.com/cdepillabout/pandoc/commit/ae4d9ffd7f8fd593fe6caa980c33fccecd187539>

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

We can try running the `pandoc` binary on our own system just to confirm it works:

```console
$ ./result/bin/pandoc --to=plain ./README.md
...
```

Here we're using Pandoc to convert the `README.md` file to a plain-text format.

Now, we can run the same binary in an Ubuntu Docker container and
prove that it does actually work on another Linux distribution:


```console
$ docker run -it \
    -v `pwd`/result/bin/pandoc:/pandoc \
    -v `pwd`/README.md:/README.md \
    ubuntu \
    /pandoc --to=plain /README.md
...
```

Great!  You can see that it outputs the same thing as when we ran it locally.
This is proof that our statically-linked `pandoc` binary is able to run
both on the system it is built on, and on any other arbitrary Linux system.
This is exactly what we were going for.

## Conclusion

`stacklock2nix` can be an easy way to get your Stack-based project to compile
with Nix.  It reuses the Nixpkgs infrastructure, so even things like static-linking
are possible.

If you've read through this series and have any problems with stacklock2nix,
feel free to [create an issue](https://github.com/cdepillabout/stacklock2nix/issues)
or [send a PR](https://github.com/cdepillabout/stacklock2nix/pulls).
