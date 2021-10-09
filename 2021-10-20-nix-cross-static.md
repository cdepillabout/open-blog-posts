------------------------------------------------------
title: Cross-compiling and Static-linking with Nix
summary: Easily cross-compile and static-link binaries using Nix
tags: nix
draft: false
------------------------------------------------------

I was having a problem at work with an ARM64 Linux edge device.  I have an
ARM64 binary on the  edge device, and I wanted to look at which dynamic
libraries it is linked to with the `readelf` program.  Unfortunately, `readelf`
is not installed on the device.  I was unable to figure out how to use the
device's package manager.

I was able to build a `readelf` binary that I could use on the device by
cross-compiling and statically-linking one from Nixpkgs.  This post
explains how to easily cross-compile and statically-link packages
from Nixpkgs.

## Nix and Nixpkgs

If you haven't heard about [Nix](https://github.com/NixOS/nix) before, it is a
programmable package manager.  It comes with a package database called
[Nixpkgs](https://github.com/NixOS/nixpkgs/).  Nixpkgs provides a large number
of common Linux packages.  One of the neat things about Nixpkgs is that since
it is programmable, it contains a few alternative, sub package sets.  These sub
package sets contain all the packages in the main package set, but compiled in
different ways.

The following sections describe different ways of compiling packages.
All of the following code is run on a NixOS x86-64 Linux machine[^1].
The Nixpkgs is channel is at commit
[`63ee5cd9`](https://github.com/NixOS/nixpkgs/commit/63ee5cd99a2e193d5e4c879feb9683ddec23fa03).

### Natively-compiled, dynamically-linked for x86-64 Linux

The following is how you build a natively-compiled `readelf` binary
dynamically-linked for an x86-64 Linux.  Note that I'm getting
`binutils-unwrapped` (which is the package that contains the `readelf` binary)
from the main package set:

```console
$ nix-build '<nixpkgs>' -A binutils-unwrapped
...
$ file ./result/bin/readelf
./result/bin/readelf: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /nix/store/gk42f59363p82rg2wv2mfy71jn5w4q4c-glibc-2.32-48/lib/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, not stripped
```

This `readelf` binary will work nicely on your machine, but its
interpreter is set to a path in the Nix store, so it is unlikely to work if you
copy it to another Linux machine[^2]:

```console
$ readelf --program-headers ./result/bin/readelf | grep interpreter
[Requesting program interpreter: /nix/store/gk42f59363p82rg2wv2mfy71jn5w4q4c-glibc-2.32-48/lib/ld-linux-x86-64.so.2]
```

(Hopefully it is not too confusing that I am using `readelf` installed on my
system to inspect the `./result/bin/readelf` binary I just built.)

### Natively-compiled, statically-linked for x86-64 Linux

The following is how you build a statically-compiled `readelf` binary for an
x86-64 Linux.  Note that I'm getting `binutils-unwrapped` from the `pkgsStatic`
sub package set.  All the binaries are statically linked in this sub package
set:

```console
$ nix-build '<nixpkgs>' -A pkgsStatic.binutils-unwrapped
...
$ file ./result/bin/readelf
./result/bin/readelf: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped
```

This `readelf` binary should mostly work if you copy it to any other x86-64
Linux system.  This is a convenient way to build packages with Nix, and
have them work on any Linux system.

### Cross-compiled, dynamically-linked for ARM64 Linux

The following is how you build a cross-compiled, dynamically-linked `readelf`
binary.  This has been cross-compiled on my x86-64 Linux machine for ARM64
Linux.  Note that this is using the `pkgsCross.aarch64-multiplatform` sub
package set.  All the binaries have been cross-compiled in this sub package
set:

```console
$ nix-build '<nixpkgs>' -A pkgsCross.aarch64-multiplatform.binutils-unwrapped
...
$ file ./result/bin/readelf
./result/bin/readelf: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /nix/store/973yqp2iwrsxi2fn2haas5q3lz091xbi-glibc-aarch64-unknown-linux-gnu-2.32-48/lib/ld-linux-aarch64.so.1, for GNU/Linux 2.6.32, not stripped
```

This has the same problem as the natively-compiled, dynamically-linked
`readelf` binary above.  Its interpreter is set to a path in the Nix store, so
it is unlikely to work if you copy it to an ARM64 Linux machine:

```console
$ readelf --program-headers ./result/bin/readelf | grep interpreter
[Requesting program interpreter: /nix/store/973yqp2iwrsxi2fn2haas5q3lz091xbi-glibc-aarch64-unknown-linux-gnu-2.32-48/lib/ld-linux-aarch64.so.1]
```

### Cross-compiled, statically-linked for ARM64 Linux

The neat thing is that these sub package sets compose, so this is how you build
a cross-compiled statically-linked `readelf` binary.  This composes
`pkgsCross.aarch64-multiplatform` with `pkgsStatic`:

```console
$ nix-build '<nixpkgs>' -A pkgsCross.aarch64-multiplatform.pkgsStatic.binutils-unwrapped
...
$ file ./result/bin/readelf
./result/bin/readelf: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, not stripped
```

It should be possible to just copy this `readelf` binary to an ARM64 Linux
device and use it like normal.

## Where to find more information

I first saw this idea of combining cross-compilation with static-linking in the
following two videos:

- [Matthew Croughan on Cross Compilation with Nix](https://www.youtube.com/watch?v=OV2hi8b5t48)
- [Nixpkgs at LLVM Distributors Conf](https://discourse.nixos.org/t/nixpkgs-at-llvm-distributors-conf/15051)

If you're just looking for more information on cross compiling, the following
page on [nix.dev](https://nix.dev) has a good introduction:

- [Cross compilation](https://nix.dev/tutorials/cross-compilation)

## Conclusion and warnings

Nix and Nixpkgs make it relatively painless to cross-compile and
statically-link binaries.  This can be convenient if you ever need
a certain program for use on a non-Nix system, even if it is a different
architecture.

Although you should be aware that the statically-linked `pkgsStatic` and
cross-compiled `pkgsCross` sub package sets do not currently have many users in
Nixpkgs.  Basic tools often will often compile (like `readelf` above), but more
complicated programs often won't compile without some work.

## Footnotes

[^1]: It is possible to use Nix on any Linux distro, not just NixOS. The
    above steps should all work on any Linux distro.

    I imagine it is unlikely to work if you are trying to follow the above
    steps with Nix on OSX.

[^2]: There is a program called [`patchelf`](https://github.com/NixOS/patchelf)
    that can be used to easily change the ELF interpreter path (dynamic loader)
    that is embedded in an executable.

    If you rewrite the interpreter path to something like `/lib/ld-linux.so.2`,
    it is possible you will be able to use the binary on a normal Linux system
    (as long as all the required dynamic libraries are also installed on
    the system).
