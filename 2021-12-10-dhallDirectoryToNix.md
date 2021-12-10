------------------------------------------------------
title: dhallDirectoryToNix
summary: Easily turn a directory of Dhall files into Nix
tags: nixos
draft: false
------------------------------------------------------

*This post is part of a series called
[The Road to `purescript2nix`](./2021-12-10-road-to-purescript2nix.md).*

I recently sent a few PRs to Dhall and Nixpkgs that add functionality for
easily reading in a directory of Dhall files into Nix.  This even works for
Dhall files that contain remote imports.

This blog post explains this new functionality, and gives pointers to the PRs
that implemented this.

The final PR that actually adds `dhallDirectoryToNix` is
<https://github.com/NixOS/nixpkgs/pull/144076>, so make sure you are using a
Nixpkgs checkout with that code before trying to use `dhallDirectoryToNix`.

## Using `dhallDirectoryToNix`

The `dhallDirectoryToNix` function operates on a directory of Dhall files.  It
evaluates the files and reads in the output as Nix.

For instance, let's take a look at an example repository that contains Dhall files:
<https://github.com/cdepillabout/example-dhall-nix>.  Clone this repository:

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
Nix REPL with Nixpkgs available:

```console
$ nix repl /some/path/to/a/local/nixpkgs/checkout/default.nix
nix-repl>
```

In the Nix REPL, call `dhallDirectoryToNix` on the above `example-dhall-nix/` directory.

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

1.  Add functionality to the `dhall-to-nixpkgs` tool that uses fixed-output
    derivations for building Dhall packages.
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

#### `dhall-to-nix`

This binary allows you to read in an arbitrary Dhall expression and convert
it to Nix.  This is used in the `dhallToNix` function in Nixpkgs.  Here's
an example of using this function in the Nix REPL.

```nix
nix-repl> dhallToNix "List/length { mapKey : Text, mapValue : Natural } (toMap { foo = 0, bar = 3})"
2
```

There are two problems with `dhallToNix`:

1.  It doesn't handle imports.  Trying to do a remote import in `dhallToNix`
    gives an error saying that remote imports are not allowed.
2.  `dhallToNix` accepts a Nix string as input.  You can't have it easily
    accept a directory of Dhall files.

#### `dhall-to-nixpkgs`

This binary converts a directory of Dhall packages to a Nix expression
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

`dhall-to-nixpkgs` is somewhat similar to the tool
[`cabal2nix`](https://github.com/NixOS/cabal2nix).
Just like Nix expressions produced by `cabal2nix` can be built
by passing them to `haskellPackages.callPackage`, Nix expressions
produced by `dhall-to-nixpkgs` can be built by passing them to
`dhallPackages.callPackage`.

The problem with this usage of `dhall-to-nixpkgs` is that the above
Nix expression takes all the remote imports in the input Dhall files as
arguments.

For example, if you look at the
[`mydhallfile.dhall`](https://github.com/cdepillabout/example-dhall-nix/blob/78f83e18fb046bfbb6a41109d9b767b84f46f425/mydhallfile.dhall#L1-L2),
you can see that it has a remote import of the following Dhall file (note
it also has an integrity check):

```dhall
let
  mkUsersList =
    https://raw.githubusercontent.com/cdepillabout/example-dhall-repo/c1b0d0327/example1.dhall
    sha256:6534a24145e93db3df3ef4bc39e2ba743404ea3e8d6cfdbb868d5c83d61f10d2

...
```

This `example-dhall-repo` becomes an argument to the function output by
`dhall-to-nixpkgs`.  This means that we have to separately package
`example-dhall-repo` by manually calling `dhall-to-nixpkgs` on it.

In theory, we shouldn't have to manually package remote imports, like
`example-dhall-repo`.  In the `mydhallfile.dhall` file, you can see that there
is an integrity check on the import
<https://raw.githubusercontent.com/cdepillabout/example-dhall-repo/c1b0d0327/example1.dhall>.
We should be able to reuse this integrity check to download this remote
import as a fixed-output derivation within Nix[^2].

The following section explains what was added to `dhall-to-nixpkgs` to
force it to turn remote Dhall imports into Nix fixed-output derivations.

### Adding a flag `--fixed-output-derivations` to `dhall-nixpkgs`

After a bit of a false start in
[Dhall PR #2304](https://github.com/dhall-lang/dhall-haskell/pull/2304)
and bunch of help from [Gabriella Gonzalez](https://www.haskellforall.com/),
I put together
[Dhall PR #2318](https://github.com/dhall-lang/dhall-haskell/pull/2318)
and [#2326](https://github.com/dhall-lang/dhall-haskell/pull/2326)
which add a new flag `--fixed-output-derivation` to the
`dhall-to-nixpkgs directory` command.

Here is an example of using this flag:

```console
$ dhall-to-nixpkgs directory --fixed-output-derivations --name "foo" --file "mydhallfile.dhall" ./.
{ buildDhallDirectoryPackage, buildDhallUrl }:
  buildDhallDirectoryPackage {
    name = "foo";
    src = ./.;
    file = "mydhallfile.dhall";
    source = false;
    document = false;
    dependencies = [
      (buildDhallUrl {
        url = "https://raw.githubusercontent.com/cdepillabout/example-dhall-repo/c1b0d0327146648dcf8de997b2aa32758f2ed735/example1.dhall";
        hash = "sha256-ZTSiQUXpPbPfPvS8OeK6dDQE6j6NbP27ho1cg9YfENI=";
        dhallHash = "sha256:6534a24145e93db3df3ef4bc39e2ba743404ea3e8d6cfdbb868d5c83d61f10d2";
        })
      (buildDhallUrl {
        url = "https://raw.githubusercontent.com/dhall-lang/dhall-lang/9758483fcf20baf270dda5eceb10535d0c0aa5a8/Prelude/List/map.dhall";
        hash = "sha256-3YRf+0Vo1AMn8qgX60LRxhOLkpynWNULwzES7zyIVoA=";
        dhallHash = "sha256:dd845ffb4568d40327f2a817eb42d1c6138b929ca758d50bc33112ef3c885680";
        })
      (buildDhallUrl {
        url = "https://raw.githubusercontent.com/dhall-lang/dhall-lang/9758483fcf20baf270dda5eceb10535d0c0aa5a8/Prelude/Text/upperASCII.dhall";
        hash = "sha256-Ra5PvYFLBHTmXCik7pKyO5eYkvpbtzcwvJlnWueQyik=";
        dhallHash = "sha256:45ae4fbd814b0474e65c28a4ee92b23b979892fa5bb73730bc99675ae790ca29";
        })
      ];
    }
```

You can see that when passing this `--fixed-output-derivations` flag, the
function produced no longer takes any arguments.  Instead, all dependencies are
packaged as fixed-output derivations using a Nix function `buildDhallUrl`.

The `hash` argument passed to `buildDhallUrl` is the Nix-compatible hash
of the Dhall file specified in the `url` argument.  This is the same as
the integrity check in the Dhall file, just base64-encoded instead of
base16-encoded.

This `--fixed-output-derivations` flag is available in `dhall-to-nixpkgs` as of
[version 1.0.7](https://hackage.haskell.org/package/dhall-nixpkgs-1.0.7).

In order to use this new `--fixed-output-derivations` flag, the `buildDhallUrl`
function will need to be present in Nixpkgs.  The next section talks about
getting that function in Nixpkgs.

### Adding `buildDhallUrl` to Nixpkgs

The `buildDhallUrl` function was added to Nixpkgs in
[Nixpkgs PR #142825](https://github.com/NixOS/nixpkgs/pull/142825).

A high-level explanation of `buildDhallUrl` is that it uses `dhall` to fetch the
remote import, and then encodes the output in a standard, Dhall-defined format.
This is done in a fixed-output derivation so that `dhall` can access the
network.  This is able to be a fixed-output derivation because of the
Dhall integrity check on the URL.

The output of `buildDhallUrl` is a standard Nixpkgs Dhall package, similar
to what is output by `dhallPackages.callPackage`. See the
[Dhall section](https://nixos.org/manual/nixpkgs/stable/#sec-language-dhall)
in the Nixpkgs manual for more info.

`buildDhallUrl` is available in Nixpkgs 21.11, and Nixpkgs `master` as of 2021-11-09.

Now that `dhall-to-nixpkgs` has the `--fixed-output-derivations` flag,
and `buildDhallUrl` is in Nixpkgs, we can write `dhallDirectoryToNix`.  The next
section explains this.

### Adding `dhallDirectoryToNix` to Nixpkgs

[Nixpkgs PR 144076](https://github.com/NixOS/nixpkgs/pull/144076) adds the
`dhallDirectoryToNix` function to Nixpkgs.

`dhallDirectoryToNix` uses
[import-from-derivation (IFD)](https://blog.hercules-ci.com/2019/08/30/native-support-for-import-for-derivation/)
to easily read in a directory of Dhall files into Nix.
Here's the example call to `dhallDirectoryToNix` again:

```nix
nix-repl> dhallDirectoryToNix { src = /some/path/to/example-dhall-nix; file = "mydhallfile.dhall"; }
[ "BILLBILLbillbill" "JANEJANEjanejane" "TESTTESTtesttest" "TESTTESTtesttest" "TESTTESTtesttest" ]
```

`dhallDirectoryToNix` roughly does the following steps:

1. Call `dhall-to-nixpkgs` on the `src` passed to `dhallDirectoryToNix`.
    This produces a Nix file corresponding to a Nixpkgs Dhall package.
2. Use IFD to import and build the Nixpkgs Dhall package produced in the
    previous step.  This uses `buildDhallUrl` under the hood.
3. Call `dhallToNix` on the resulting Nixpkgs Dhall package in the previous step.
    This evaluates the Dhall package built in the previous step and converts
    it to Nix code.

Check out the above PR if you're interested in exactly how this works.

`dhallDirectoryToNix` is available in Nixpkgs `master` as of
2021-12-08. It will likely be available in Nixpkgs 22.05.

## Caveats

There are two problems with `dhallDirectoryToNix` you might run into:

1.  `dhall-to-nix` can't convert _all_ Dhall expressions into Nix,
    so you're not able to convert any arbitrary Dhall expression to Nix.
    But `dhall-to-nix` does seem good at converting JSON-like Dhall expressions
    to Nix.  I imagine most Dhall files that people want to read into
    Nix are basic JSON-like expressions.
2.  `dhallDirectoryToNix` doesn't seem to work well when using it from a
    flake.  You can find out more in
    [this related issue](https://github.com/cdepillabout/purescript2nix).

## Conclusion

Implementing `dhallDirectoryToNix` ended up being harder than I was expecting,
but came out quite nice.  `dhallDirectoryToNix` is an easy way to read an
arbitrary directory of Dhall files in as a Nix expression.  It even supports
remote imports in your Dhall files (as long as they have integrity checks).

## Footnotes

[^1]: `dhall-to-nixpkgs` has other functionality as well, but it
    is not relevant to the above explanation.  See
    the [Dhall section](https://nixos.org/manual/nixpkgs/stable/#sec-language-dhall)
    in the Nixpkgs manual for more info.

[^2]: I took a quick look, but I couldn't find a succinct explanation of a
    fixed-output derivation anywhere on the net.  Basically, it is a derivation
    where you know in advance the hash of what will be output after building
    the derivation.  These derivations are treated specially by Nix.  You're
    able to access the network while building these derivations.  Fixed-output
    derivations are normally used for downloading files from the internet.
