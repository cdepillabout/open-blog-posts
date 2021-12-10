------------------------------------------------------
title: PureScript Package Sets with Hashes
summary: PureScript package sets with hashes for consumption by Nix
tags: purescript
draft: false
------------------------------------------------------

*This post is part of a series called
[The Road to `purescript2nix`](./2021-12-10-road-to-purescript2nix.md).*

The [PureScript package sets](https://github.com/purescript/package-sets)
are Dhall files that specify a group of PureScript packages that are known to all
compile together.  If you're coming from Haskell, this is similar to a
[Stackage resolver](https://www.stackage.org/).

Package sets make development quite easy, since you have a large number of
libraries that you know all work together.  You don't have to worry about
manually figuring out what versions of packages to use in order to successfully
build.

The one big problem with PureScript's package sets are that they don't
include hashes for packages.  This blog post describes how I created a
proof-of-concept package set that contains hashes, and how it can be used.

## Package Sets

Here's an example of two packages from a PureScript package set:

```dhall
{ abides =
  { dependencies = [ "enums", "foldable-traversable" ]
  , repo = "https://github.com/athanclark/purescript-abides.git"
  , version = "v0.0.1"
  }
, ace =
  { dependencies =
    ["arrays", "effect", "foreign", "nullable", "web-html", "web-uievents"]
  , repo = "https://github.com/purescript-contrib/purescript-ace.git"
  , version = "v8.0.0"
  }
, ...
}
```

The two packages are `abides` and `ace`.  You can see that each package has a
list of dependencies, a repository, and an associated Git tag (`version`).

I created an
[RFC for adding a `hash` field](https://github.com/purescript/package-sets/issues/1042)
to each package, as well as a
[PR adding a script for computing hashes](https://github.com/purescript/package-sets/pull/1043).

The script for computing hashes takes an existing package set (like the example
above), and produces a package set that looks like this:

```dhall
{ abides =
  { dependencies = [ "enums", "foldable-traversable" ]
  , hash = "sha256-nrZiUeIY7ciHtD4+4O5PB5GMJ+ZxAletbtOad/tXPWk="
  , repo = "https://github.com/athanclark/purescript-abides.git"
  , version = "v0.0.1"
  }
, ace =
  { dependencies =
    ["arrays", "effect", "foreign", "nullable", "web-html", "web-uievents"]
  , hash = "sha256-cALaq9mxO9s6WDtDpydyZIHiM+IPa1xGGUtgS2A/qWA="
  , repo = "https://github.com/purescript-contrib/purescript-ace.git"
  , version = "v8.0.0"
  }
, ...
}
```

You can see that this is the same as above, but now each package has a `hash`
field as well.  This would allow you to download the source of the package
from GitHub and check that that the hashes match.

See the above RFC and PR for implementation details.

## Future of PureScript package sets

It seems like development has completely moved to the
[PureScript Registry](https://github.com/purescript/registry) instead of the
PureScript package sets.  The PureScript Registry is somewhat similar to the
idea of package sets, but more flexible and featureful.

The above RFC and PR were not accepted on the grounds that new features won't be
added to the package sets repository moving forward.  Also, the PureScript Registry will contain hashes
for package versions.  It doesn't sound like there is a firm release date for
the Registry, but Fabrizio says that they'd like to
[get the Registry working sometime in December 2021](https://github.com/purescript/package-sets/issues/1042#issuecomment-981626792).

However, if you'd like to get old PureScript package sets buildable with a system
that requires hashed inputs (like Nix), you may want to take a look at the
this issue: <https://github.com/cdepillabout/purescript2nix/issues/4>

## Conclusion

It is unfortunate that this RFC didn't go through, but at least now there is
a script to easily add hashes to a PureScript package set.  It is also great
that the PureScript Registry will have hashes from the very beginning.
This will hopefully make it much easier to consume PureScript packages from
a build system that requires inputs to be hashed.
