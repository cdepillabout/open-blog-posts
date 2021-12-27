------------------------------------------------------
title: Getting Started with PureNix
summary: How to setup an environment for playing around with PureNix
tags: haskell, nixos, purescript
draft: false
------------------------------------------------------

*This post is the first post in a
[series about PureNix](./2021-12-26-purenix).*

[PureNix](https://github.com/purenix-org/purenix) is a Nix backend for
[PureScript](https://www.purescript.org/).  PureNix allows you to write
PureScript code and transpile it to Nix code.

This post gives an overview of PureScript, PureNix, and explains how to get
started using PureNix.

## PureScript

[PureScript](https://en.wikipedia.org/wiki/PureScript) is a strongly-typed,
pure functional programming language. Here's a small example of a function
written in PureScript:

```purescript
module Main where

import Prelude

type FilePath = String

subdirectory :: FilePath -> FilePath -> FilePath
subdirectory p1 p2 = p1 <> "/" <> p2
```


## Conclusion

*This post is the first post in a
[series about PureNix](./2021-12-26-purenix).*

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
