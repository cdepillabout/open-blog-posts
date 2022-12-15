------------------------------------------------------
title: stacklock2nix
summary: Easily use Nix to build a Haskell package with a `stack.yaml.lock` file
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
you have a `stack.yaml` and `stack.yaml.lock` file).

You can find usage instructions, as well as two example projects in the
[README](https://github.com/cdepillabout/stacklock2nix#readme).

I've decided to do a series of blog posts where I use `stacklock2nix` to package
various Haskell projects.  Check out each one for a realistic example of using
`stacklock2nix`:

1. `purescript` (coming soon)

2. `dhall` (coming soon)

3. `spago` (coming soon)


## Footnotes

[^1]: haskell.nix of course has a bunch of
    [good points](https://github.com/cdepillabout/stacklock2nix#stacklock2nix-vs-haskellnix) as well.

