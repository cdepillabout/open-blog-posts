------------------------------------------------------
title: purescript2nix
summary: Easily build a PureScript project with Nix
tags: nix
draft: false
------------------------------------------------------

*This post is part of a series called
[The Road to `purescript2nix`](./2021-12-20-road-to-purescript2nix.md).*


## Using `dhallDirectoryToNix`

## Conclusion

## Footnotes

[^1]: `dhall-to-nixpkgs` has other functionality as well, but it
    is not relevant to the above explanation.  See
    the [Dhall section](https://nixos.org/manual/nixpkgs/stable/#sec-language-dhall)
    in the Nixpkgs manual for more info.

[^2]: I took a quick look, but I couldn't find an succinct explanation of what
    a fixed-output derivation is.  Basically, it is a derivation where you know
    in advance the hash of what will be output after building the derivation.
    These derivations are treated specially by Nix.  You're able to do network
    access during these derivations.  Fixed-output derivations are normally
    used for downloading files from the internet.

