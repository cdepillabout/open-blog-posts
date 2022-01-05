------------------------------------------------------
title: Git no_push
summary: Setup a Git remote so you can't accidentally push to it
tags: nixos
draft: false
------------------------------------------------------

This is a trick I learned while working in
[Nixpkgs](https://github.com/NixOS/nixpkgs).
It is possible to add a Git remote repository and set it up so you can't easily
push to it.  After cloning the repository, run the following command:

```console
$ git remote set-url --push origin no_push
```

This makes it so that Git will fail if you try to push to `origin`.
Here's what the `git remote` output looks like, using Nixpkgs as an example:

```console
$ git remote -v
origin  git@github.com:NixOS/nixpkgs.git (fetch)
origin  no_push (push)
```

Here's what the corresponding entry in `.git/config` looks like:

```console
$ cat .git/config
...
[remote "origin"]
	url = git@github.com:NixOS/nixpkgs.git
	fetch = +refs/heads/*:refs/remotes/origin/*
	pushurl = no_push
...
```

This is useful when you have access to a repository, but you almost never want
to push branches directly to it.  You almost always want to interact with it by
sending PRs from your own fork.

This is how most Nixpkgs committers interact with Nixpkgs.  Almost all changes
to Nixpkgs come through PRs.  Very few changes are pushed directly to Nixpkgs.
It is relatively uncommon for Nixpkgs committers to have a branch live in the
Nixpkgs repository itself.

If you've setup a Git remote using this `no_push` trick, but you _do_ need to
push to a branch in Nixpkgs, you can force the correct remote to be used with
a command like this:

```console
$ git push -v git@github.com:NixOS/nixpkgs.git HEAD
```

You can also explicitly set the `pushRemote` for a single branch.  This will
allow you to directly push that given branch (but no other branches).  The
easiest way to set this up is directly through the `.git/config` file.  Add an
entry for the branch that looks similar to the following:

```console
$ cat .git/config
...
[branch "haskell-updates"]
	remote = origin
	pushRemote = git@github.com:NixOS/nixpkgs.git
	merge = refs/heads/haskell-updates
...
```

This makes it so that when the `haskell-updates` branch is checked out, you can
run `git push` and it automatically pushes to `git@github.com:NixOS/nixpkgs.git`.
