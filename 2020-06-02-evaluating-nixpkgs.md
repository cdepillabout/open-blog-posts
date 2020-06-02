------------------------------------------------------
title: How to Make Sure Nixpkgs Can Evaluate
summary: Explain a command to make sure nixpkgs is capable of evaluating
tags: nixos
draft: false
------------------------------------------------------

When sending pull requests to Nixpkgs, sometimes the
[`ofborg`](https://github.com/NixOS/ofborg) CI will display
evaluation errors. This blog post explains how to reproduce these
errors locally for debugging.

## Example Error

Here is an [example PR](https://github.com/NixOS/nixpkgs/pull/88426) where
`ofborg` reports an evaluation error.

Here is the evaluation
[error](https://gist.github.com/GrahamcOfBorg/804dff1aaa3f13aed3a6999c8043a42f)
that `ofborg` is reporting:

```
nix-env failed:
warning: ignoring the user-specified setting 'restrict-eval', because it is a restricted setting and you are not a trusted user
error: while evaluating anonymous function at .gc-of-borg-outpaths.nix:44:12, called from undefined position:
while evaluating anonymous function at pkgs/top-level/release-lib.nix:121:6, called from lib/attrsets.nix:292:43:
while evaluating 'hydraJob' at lib/customisation.nix:171:14, called from pkgs/top-level/release-lib.nix:121:14:
while evaluating the attribute 'drvPath' at lib/customisation.nix:188:13:
while evaluating the attribute 'drvPath' at lib/customisation.nix:155:13:
while evaluating the attribute 'buildInputs' of the derivation 'blake2-0.3.0' at pkgs/development/haskell-modules/generic-builder.nix:289:3:
while evaluating the attribute 'propagatedBuildInputs' of the derivation 'hlint-3.1.1' at pkgs/development/haskell-modules/generic-builder.nix:289:3:
while evaluating 'getOutput' at lib/attrsets.nix:464:23, called from undefined position:
while evaluating anonymous function at pkgs/stdenv/generic/make-derivation.nix:157:17, called from undefined position:
attribute 'ghc-lib-parser_8_10_1_20200412' missing, at pkgs/development/haskell-modules/configuration-common.nix:1510:22
```

This exact error is not important[^1]. In the next section I will explain how
to replicate this CI check locally.

## Replicating This Error Locally

In order to replicate this CI check in your local environment, first you'll
have to checkout the commit in nixpkgs where the error is occurring.

The base of the [above PR](https://github.com/NixOS/nixpkgs/pull/88426) is
[f043b2c3](https://github.com/NixOS/nixpkgs/commit/f043b2c31bb89fd2024883e348deb994d98ab8a9),
so we clone nixpkgs and checkout that commit:

```console
$ git clone git@github.com:NixOS/nixpkgs.git
$ cd nixpkgs
$ git checkout f043b2c3
```

Now we can try evaluating nixpkgs in order to reproduce this error:

```console
$ nix-env --query --available --out-path --file ./. --show-trace
error: while querying the derivation named 'hlint-3.1.1':
while evaluating the attribute 'buildInputs' of the derivation 'hlint-3.1.1' at pkgs/development/haskell-modules/generic-builder.nix:289:3:
while evaluating 'getOutput' at lib/attrsets.nix:464:23, called from undefined position:
while evaluating anonymous function at pkgs/stdenv/generic/make-derivation.nix:143:17, called from undefined position:
attribute 'ghc-lib-parser_8_10_1_20200412' missing, at pkgs/development/haskell-modules/configuration-common.nix:1510:22
```

This `nix-env` command is the take away from this blog post:

`nix-env --query --available --out-path --file ./. --show-trace`.

I'm not sure why the `--out-path` flag is needed, but if you don't add it then
you don't get this error.

## Conclusion

The above `nix-env` command is a convenient way to make sure there are no
evaluation errors in nixpkgs, or to figure out where an evaluation error is
coming from.

I picked up this trick from [@LnL7](https://github.com/LnL7) in
[this PR](https://github.com/NixOS/nixpkgs/pull/89308).

## Footnotes

[^1]: This error is saying that the `ghc-lib-parser_8_10_1_20200412` derivation
    is used, but it is never defined anywhere.

    This is caused because of how Haskell packages are updated automatically in
    the `haskell-updates` branch.  See
    [this](https://discourse.nixos.org/t/haskellpackages-stm-containers-fails-to-build/5416/4)
    for more information.

    Note that this CI error is not directly related to the
    [PR](https://github.com/NixOS/nixpkgs/pull/88426) in question.  The
    underlying `haskell-updates` branch has a problem, which is what the
    `ofborg` CI is saying.
