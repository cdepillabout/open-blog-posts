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

```nix
nix-repl> dhallDirectoryToNix { src = /some/path/to/example-dhall-nix; file = "mydhallfile.dhall"; }
[ "BILLBILLbillbill" "JANEJANEjanejane" "TESTTESTtesttest" "TESTTESTtesttest" "TESTTESTtesttest" ]
```

This shows how the output of the Dhall file is now available for us to use
within Nix.  This functionality can be really helpful when you have information
in Dhall files that is needed in Nix to be able to make decisions about how to
build packages.

## Implementing `dhallDirectoryToNix`

Implementing `dhallDirectoryToNix` happened in a few stages:

1.  Add functionality to the `dhall-to-nixpkgs` tool for producing Nix code that
    uses fixed-output derivations for building Dhall packages.
2.  The above change relies on a `buildDhallUrl` Nix function, so get that in
    Nixpkgs.
3.  Send the actual PR adding `dhallDirectoryToNix` to Nixpkgs.

The following sections talk about each of these changes.

### `dhall-to-nix` and `dhall-to-nixpkgs`


Dhall contains two binaries related to using Dhall with Nix:
[`dhall-to-nix`](https://hackage.haskell.org/package/dhall-nix) (from the
`dhall-nix` package on Hackage) and
[`dhall-to-nixpkgs`](https://hackage.haskell.org/package/dhall-nixpkgs)
(from the `dhall-nixpkgs` package on Hackage).

-   `dhall-to-nix`

    This binary allows you to read in an arbitrary dhall expression and convert
    it to Nix.  This is used in the `dhallToNix` function in Nixpkgs.  Here's
    an example of using this function in the Nix repl.

    ```nix
    nix-repl> dhallToNix "List/length { mapKey : Text, mapValue : Natural } (toMap { foo = 0, bar = 3})"
    2
    ```

    There are two problems with `dhallToNix`:

    1.  It doesn't handle things like imports.  Trying to do a remote import in
        `dhallToNix` gives you an error saying that remote imports are not
        allowed.
    2.  `dhallToNix` accepts a Nix string as input.  You can't have it easily
        accept a directory of Dhall files.

-   `dhall-to-nixpkgs`

    This binary converts a directory of Dhall packages to to a Nix expression
    for building them in Nixpkgs[^1].  Here is an example of running
    `dhall-to-nixpkgs` on the above example package:

    ```console
    $ dhall-to-nixpkgs directory --name "foo" --file "mydhallfile.dhall" ./.
    { buildDhallDirectoryPackage, example-dhall-repo, Prelude }:
      buildDhallDirectoryPackage {
        name = "foo";
        src = ./.;
        file = "mydhallfile.dhall";
        source = false;
        document = false;
        dependencies = [
          (example-dhall-repo.overridePackage { file = "example1.dhall"; })
          (Prelude.overridePackage { file = "List/map.dhall"; })
          (Prelude.overridePackage { file = "Text/upperASCII.dhall"; })
          ];
        }
    ```

    This produces an expression that can be built by passing it to
    `dhallPackages.callPackage` in Nixpkgs.

    `dhall-to-nixpkgs` is somewhat similar to the tool `cabal2nix`.
    Just like Nix expressions produced by `cabal2nix` can be built
    by passing them to `haskellPackages.callPackage`, Nix expresssions
    produced by `dhall-to-nixpkgs` can be built by passing them to
    `dhallPackages.callPackage`.

    The problem with this usage of `dhall-to-nixpkgs` is that the above
    Nix expression that has been produced takes all the remote imports
    in the input Dhall files as arguments.

    For example, if you look at the
    [`mydhallfile.dhall`](https://github.com/cdepillabout/example-dhall-nix/blob/78f83e18fb046bfbb6a41109d9b767b84f46f425/mydhallfile.dhall#L1-L2),
    you can see that it has a remote import of the following Dhall file.  It
    also has an integrity check:

    ```dhall
    let
      mkUsersList =
        https://raw.githubusercontent.com/cdepillabout/example-dhall-repo/c1b0d0327/example1.dhall
        sha256:6534a24145e93db3df3ef4bc39e2ba743404ea3e8d6cfdbb868d5c83d61f10d2

    ...
    ```

    This `example-dhall-repo` becomes an argument to the expression output by
    `dhall-to-nixpkgs`.  This means that we have to separately package
    `example-dhall-repo` by manually calling `dhall-to-nixpkgs` on it.
    Although in theory, we shouldn't have to.  In the `mydhallfile.dhall` file,
    you can see that there is an integrity check on the import
    <https://raw.githubusercontent.com/cdepillabout/example-dhall-repo/c1b0d0327/example1.dhall>.
    We should be able to reuse this integrity check to download this remote
    import as a fixed-output derivation[^2].

    The following section explains what was added to `dhall-to-nixpkgs` to
    force it to turn remote Dhall imports into Nix fixed-output derivations.

### Adding a flag to `dhall-nixpkgs`

https://github.com/dhall-lang/dhall-haskell/pull/2304
https://github.com/dhall-lang/dhall-haskell/pull/2318
https://github.com/dhall-lang/dhall-haskell/pull/2326

### Adding `buildDhallUrl` to Nixpkgs

https://github.com/NixOS/nixpkgs/pull/142825

### Adding `dhallDirectoryToNix` to Nixpkgs

https://github.com/NixOS/nixpkgs/pull/144076

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
