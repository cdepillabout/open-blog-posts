------------------------------------------------------
title: Get Involved with Nixpkgs Haskell
summary: Explanation on how to get started with contributing to the Haskell subsystem of Nixpkgs
tags: nixos, haskell
draft: false
------------------------------------------------------

This post explains how you can get started with contributing to the Haskell
subsystem of Nixpkgs.  This will mainly be helpful for people who are happily
using Haskell binaries and Haskell libraries from Nixpkgs, but want to start
contributing additional fixes, or helping the Haskell subsystem in Nixpkgs
advance faster.

This post lists a few different ways to get involved, from easiest to hardest.

## Method 1: Join our Matrix Channel

The easiest way to get involved is to join the Matrix channel for the Haskell
infrastructure in Nixpkgs:
[#haskell:nixos.org](https://matrix.to/#/#haskell:nixos.org).  Here you can talk
to the Haskell maintainers in Nixpkgs, as well as other users.  This is a great place
to ask casual questions.

## Method 2: Answer Haskell- and Nix-related Questions on Other Sites

People often ask questions related to the Haskell subsystem of Nixpkgs on other
websites. The maintainers in Nixpkgs are not always watching these other
websites, so it is very helpful for other members of the community to answer
questions.

Non-exhaustive list of other sites where people frequently ask questions:

- [NixOS Discourse](https://discourse.nixos.org/)
- [Unofficial NixOS Discord Server](https://discord.com/invite/RbvHtGa)
- [/r/haskell](https://www.reddit.com/r/haskell)
- [`#haskell`](https://twitter.com/hashtag/haskell),
    [`#nix`](https://twitter.com/hashtag/nix), and
    [`#nixos`](https://twitter.com/hashtag/nixos) tags on Twitter

## Method 3: Add Yourself to the Maintainers List for Your Haskell Packages in Nixpkgs

Haskell packages in Nixpkgs have a notion of a maintainer that is specific to
Nixpkgs.  The maintainer in Nixpkgs could be different from the person who
uploads the packages to Hackage.  When Haskell packages break in Nixpkgs, the
maintainers are notified on GitHub.
[Here's](https://github.com/NixOS/nixpkgs/pull/199424#issuecomment-1305869454)
an example of a notification.  You can see that a couple well-known packages
were broken, and the maintainers were pinged.

If you add yourself to the maintainer list for a given Haskell package, you'll
be pinged on GitHub when it breaks and stops compiling.  You'll have time to
send a PR fixing it before Nixpkgs moves on with the broken package.

To add yourself to the maintainers list for specific packages, send a PR
adding your GitHub handle and a list of Haskell packages
[here](https://github.com/NixOS/nixpkgs/blob/3e1a3aa9392fd2be39717eb1ea2a08253d1ba3fd/pkgs/development/haskell-modules/configuration-hackage2nix/main.yaml#L166).

Again, you can add yourself as a maintainer for any Haskell package you use.
You don't have to be the original author of the package.

## Method 4: Watch the `haskell` Label in Nixpkgs

Haskell-related issues and PRs get tagged with the [`haskell`
label](https://github.com/NixOS/nixpkgs/labels/6.topic%3A%20haskell) in
Nixpkgs.  Responding to issues and PRs can take a lot of work, so we
really appreciate when other users help with responding to issues, or do
reviews of PRs.

Even if you feel like you're not knowledgable enough with how the Haskell ecosystem
works in Nixpkgs, we still really appreciate comments like "I can confirm
this is a problem, it fails for me too!", or "I've tried building the
derivation from this PR and I can confirm it works!"

After reviewing even 5 to 10 Haskell PRs, you should have a good idea of the
common sources of problems, and how we fix them.  You'll be able to make
suggestions to new PRs without us needing to be involved. It is so much easier
for us to just be able to click the "Merge" button on GitHub than having to go
back and forth on a PR.

## Method 5: Send PRs Fixing Broken Haskell Packages in Nixpkgs

Every week or two In Nixpkgs, we branch off `master` with the
`haskell-updates` branch and update our Stackage and Hackage pins. This often
causes some Haskell packages to break.  We spend a couple days trying to fix as
much of the breakage as possible.  When the `haskell-updates` branch is in a
good state, we merge it back in to `master` and repeat the process.
[Here](https://github.com/NixOS/nixpkgs/pull/199424) is an example of one of
these `haskell-updates` PRs. The specific details of this process are
documented in
[`HACKING.md`](https://github.com/NixOS/nixpkgs/blob/b457130e8a21608675ddf12c7d85227b22a27112/pkgs/development/haskell-modules/HACKING.md)
in the Nixpkgs repo.

We have a [daily
build-report](https://github.com/cdepillabout/nix-haskell-updates-status) that
shows which packages are broken.  We always appreciate when people watch this
build report, and send fixes for broken packages.  This _really_ helps us be
able to merge the `haskell-updates` branch faster, and get the most up-to-date
libraries and compilers out to our users.  If you've ever heard someone
complain about Nixpkgs not having the latest GHC, you should really point them
to this blog post and ask them to help out![^1]

In theory, all you should need to do is watch this daily build-report, but in
practice, you will likely also want to watch the [`haskell-updates` jobset on
Hydra](https://hydra.nixos.org/jobset/nixpkgs/haskell-updates#tabs-errors), and
fix any evaluation errors that are present.  These also prevent us from merging
in the `haskell-updates` branch to `master`.

## Conclusion

If you're a happy Haskell and Nixpkgs user (or even better, an _un_happy user),
we'd really appreciate help keeping Nixpkgs in good shape.  Please feel free to
join us on Matrix, or dive in and start sending PRs!

## Footnotes

[^1]: To add a little more explanation to this, here's the general route for
    end-users to be able to use Haskell-related changes in Nixpkgs:

    1. We ask that all Haskell-related changes in Nixpkgs to be sent as PRs to the
        `haskell-updates` branch. This includes things like new GHC versions.
        We review PRs to the `haskell-updates` branch and merge them in.
    1. The Hydra [jobset for `haskell-updates`](https://hydra.nixos.org/jobset/nixpkgs/haskell-updates)
        builds all Haskell-related packages.  We can merge the
        `haskell-updates` branch into `master` as soon as CI is clean.
    1. After merging `haskell-updates` into `master`, the various Hydra jobsets for
        the `nixpkgs-unstable` and `nixos-unstable` channels go to work.  Once
        these are finished, end-users are easily able to use new
        Haskell packages and compilers.
    1. We restart this process.
