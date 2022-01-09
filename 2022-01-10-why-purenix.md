------------------------------------------------------
title: Why PureNix?
summary: What lead to us starting to write PureNix?
tags: haskell, nixos, purescript
draft: false
------------------------------------------------------

*This post is the third post in a [series about PureNix](./2022-01-03-purenix).
The previous post is about
[who would find PureNix easy to use](./2022-01-05-who-would-like-purenix).*

[PureNix](https://github.com/purenix-org/purenix) started out half as a joke.
This post explains why we started working on PureNix, and how it moved from a
joke to something we are excited about.

## The Idea

I was a mentor for [Summer of Nix 2021](https://summer.nixos.org/).  One of the
nice things about Summer of Nix is that the organizers arranged for various
people in the Nix community to give presentations about Nix-related topics.
One of the presentations was from [@adisbladis](https://github.com/adisbladis)
about [poetry2nix](https://github.com/nix-community/poetry2nix).

`poetry2nix` is a Nix library for building Python packages that use
[Poetry](https://python-poetry.org/) as a build system.  Poetry is a build tool
like Haskell's `stack`/`cabal`, Rust's `cargo`, etc.  Python packages using
Poetry must have a `pyproject.toml` file.  This file is similar to Haskell's
`.cabal` file, Rust's `Cargo.toml` file, etc.  Here's an
[example `pyproject.toml`](https://github.com/michaeloliverx/python-poetry-docker-example/blob/f7241bf6586e99c6c649eba36ca0efd935ea6316/pyproject.toml)
file. If you look at the example `pyproject.toml` file, you can see there are many
things you'd expect in a project configuration file, like dependencies (and
specified versions).

Running Poetry produces a `poetry.lock` file, which locks all dependencies and
transitive dependencies to specific versions.  This lock file also contains
hashes for their source code.  The `poetry.lock` file is in
[TOML format](https://en.wikipedia.org/wiki/TOML).  This `poetry.lock` file is
similar to Haskell's `stack.yaml.lock` file, Rust's `Cargo.lock` file, etc.

`poetry2nix` works by using Nix's `builtins.fromTOML` to read the `poetry.lock` and
`pyproject.toml` files in as Nix values.


## Conclusion

*This post is the third post in a [series about PureNix](./2022-01-03-purenix).
The previous post is about
[who would find PureNix easy to use](./2022-01-05-who-would-like-purenix).*

## Using PureNix Professionally

If your company is considering PureNix in order to tame a complicated Nix
codebase, [Jonas](https://jonascarpay.com/) and I currently have time available
for consulting or freelance.  Feel free to [get in touch](/about).  We are also
available for any other Nix/Haskell/PureScript-related work.
