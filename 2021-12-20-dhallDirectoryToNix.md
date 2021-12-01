------------------------------------------------------
title: dhallDirectoryToNix
summary: Easily turn a directory of Dhall files into Nix
tags: nix
draft: false
------------------------------------------------------

I recently sent a few PRs to Dhall and Nixpkgs that add functionality for
easily reading in a directory of Dhall files into Nix.

This blog post explains this new functionality, and gives pointers to the PRs
that implemented this.

The final PR that actually adds `dhallDirectoryToNix` is
<https://github.com/NixOS/nixpkgs/pull/144076>, so make sure you are using a
Nixpkgs checkout with that code before trying to use `dhallDirectoryToNix`.

## Using `dhallDirectoryToNix`

The `dhallDirectoryToNix` function operates on a directory of Dhall files.  It
evaluates the files and reads in the output as Nix.

For instance, let's take a look at an example repo that contains Dhall files:
<https://github.com/cdepillabout/example-dhall-nix>.  Clone this repo:

```console
$ git clone https://github.com/cdepillabout/example-dhall-nix
$ cd example-dhall-nix/
```

Let's first use Dhall to evaluate the `mydhallfile.dhall` file:

```console
$ dhall < ./mydhallfile.dhall
[ "BILLBILLbillbill"
, "JANEJANEjanejane"
, "TESTTESTtesttest"
, "TESTTESTtesttest"
, "TESTTESTtesttest"
]
```

If you look through the `mydhallfile.dhall` file, you'll see it contains both
local imports and remote imports.  All remote imports are protected with
integrity checks.  Evaluating this file produces a list of strings.

We can read this file into Nix using `dhallDirectoryToNix`.  First, get into a
Nix repl with Nixpkgs available:

```console
$ nix repl /some/path/to/a/local/nixpkgs/checkout/default.nix
nix-repl>
```

In the Nix repl, call `dhallDirectoryToNix` on the above `example-dhall-nix/` directory.

```console
nix-repl> dhallDirectoryToNix { src = /some/path/to/example-dhall-nix; file = "mydhallfile.dhall"; }
[ "BILLBILLbillbill" "JANEJANEjanejane" "TESTTESTtesttest" "TESTTESTtesttest" "TESTTESTtesttest" ]
```

This shows how the output of the Dhall file is now available for us to use
within Nix.  This functionality can be really helpful when you have information
in Dhall files that is needed in Nix to be able to make decisions about how to
build packages.

## Implementing `dhallDirectoryToNix`

Implementing `dhallDirectoryToNix` happened in a few stages:

1. Add functionality to the `dhall-nixpkgs` tool for producing Nix code that
    uses fixed-output derivations for building Dhall packages.
2. The above change relies on a `buildDhallUrl` Nix function, so get that in
    Nixpkgs.
3. Send the actual PR adding `dhallDirectoryToNix` to Nixpkgs.

The following sections talk about each of these changes.

### `dhall-to-nix` and `dhall-to-nixpkgs`


Dhall contains two binaries related to using Dhall with Nix:
[`dhall-to-nix`](https://hackage.haskell.org/package/dhall-nix) (from the
`dhall-nix` package on Hackage) and
[`dhall-to-nixpkgs`](https://hackage.haskell.org/package/dhall-nixpkgs)
(from the `dhall-nixpkgs` package on Hackage).

- `dhall-to-nix`

    This binary allows you to read in a 

### Adding flag to `dhall-nixpkgs`

### Adding `buildDhallUrl` to Nixpkgs

### Adding `dhallDirectoryToNix` to Nixpkgs
