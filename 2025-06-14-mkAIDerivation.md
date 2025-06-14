------------------------------------------------------
title: mkAIDerivation
summary: Generate a Nix derivation on the fly using an LLM over the network
tags: nixos
draft: false
------------------------------------------------------

I just released
[`mkAIDerivation`](https://github.com/cdepillabout/mkAIDerivation).  This is a
Nix function that let's you generate a derivation on the fly using an LLM over
the network.  Here's what it looks like:

```nix
mkAIDerivation ''
    Generate a derivation for building `xterm`
''
```

This uses [evil-nix](https://github.com/cdepillabout/evil-nix) to be able to
(im)purely do network access without a hash.

Check out the [`mkAIDerivation` repo](https://github.com/cdepillabout/mkAIDerivation)
for more information.
