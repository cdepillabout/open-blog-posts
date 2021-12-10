------------------------------------------------------
title: The Road to purescript2nix
summary: Steps to get to purescript2nix
tags: nix
draft: false
------------------------------------------------------

I recently put together a tool called
[`purescript2nix`](https://github.com/cdepillabout/purescript2nix).
This is a Nix function that lets you easily build a PureScript project with
Nix.

There were a couple steps along the way, and this blog post series details all
the steps.  If you're only interested in using `purescript2nix`, jump directly
to the last post.

1.  [`dhallDirectoryToNix`](./2021-12-20-dhallDirectoryToNix)

    This post explains the `dhallDirectoryToNix` function, which is used for
    reading a directory of Dhall files in as Nix code.

2.  [PureScript Package Sets with Hashes](./2021-12-20-purescript-package-set-with-hashes)

    This post explains a method of adding a hash for each package to a
    PureScript package set.

3.  [`purescript2nix`](./2021-12-20-purescript2nix.md)

    This post explains how to use `purescript2nix`.
