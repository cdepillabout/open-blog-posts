------------------------------------------------------
title: stacklock2nix
summary: Easily use Nix to build a Haskell package with a stack.yaml.lock file
tags: nixos, haskell
draft: false
------------------------------------------------------

For a while now I've wanted a way to use Nix to build a Haskell package
that has a `stack.yaml` file.  The go-to method in the Haskell community is to
use [haskell.nix](https://github.com/input-output-hk/haskell.nix), but
haskell.nix has a few downsides[^1]:

-   Evaluation can take quite a while, and building your project without using
    the IOHK cache can take a very long time.
-   haskell.nix is quite complicated.  It can be hard to figure out problems
    and fix bugs.
-   It is completely separate from the Haskell infrastructure in Nixpkgs.

I decided to write a Nix library called
[`stacklock2nix`](https://github.com/cdepillabout/stacklock2nix).  It generates
a Nixpkgs-compatible Haskell overlay from a `stack.yaml` and `stack.yaml.lock`
file.  This allows you to easily build a Haskell project with Nix (as long as
you have a `stack.yaml` and `stack.yaml.lock` file).  It allows you to use
your `stack.yaml` file as a single-source-of-truth for Haskell dependency versions[^2].

You can find usage instructions, as well as two example projects in the
[README](https://github.com/cdepillabout/stacklock2nix#readme).

I've decided to do a series of blog posts where I use `stacklock2nix` to package
various Haskell projects.  Check out each one for a realistic example of using
`stacklock2nix`:

1. [PureScript](./2022-12-16-building-purescript-with-stacklock2nix)

    This post introduces an easy way to build a straight-forward Haskell
    project with `stacklock2nix`.  This is good for beginners.

2. [Dhall](./2022-12-20-building-dhall-with-stacklock2nix)

    This post introduces an advanced way to build a Haskell project with
    multiple packages.  This is good for developers who need ultimate
    flexibility.

3. [Pandoc](./2022-12-26-building-pandoc-with-stacklock2nix)

    This post is similar to the post about building the PureScript compiler,
    but it sets up Pandoc to be statically linked.  This would be a good
    example to follow for people that want to distribute fully
    statically-linked Haskell binaries.

## Footnotes

[^1]: haskell.nix of course has a bunch of
    [good points](https://github.com/cdepillabout/stacklock2nix#stacklock2nix-vs-haskellnix) as well.

[^2]: As opposed to manually keeping dependency versions in sync between
    `stack.yaml` and your Nix code.

