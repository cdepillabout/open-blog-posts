------------------------------------------------------
title: How to Setup LXD on NixOS
summary: Explanation on how to setup LXD on NixOS with a NixOS guest using unmanaged network bridge.
tags: nixos
draft: false
------------------------------------------------------

I put together an explanation of how to setup LXD on a NixOS host with a NixOS guest that uses a unmanaged bridge for networking:

<https://discourse.nixos.org/t/howto-setup-lxd-on-nixos-with-nixos-guest-using-unmanaged-bridge-network-interface/21591?u=cdepillabout>


<!-- Here's the full text of the Discourse post in case if ever gets taken down or something. -->

<!--

I recently setup LXD on my NixOS machine in order to run a guest NixOS container.

LXD has a lot of configuration options, and it is sometimes difficult to figure out the right setup for your use-case.  Networking is especially complex.  LXD has support for many different types of networking setups.  By default, LXD pushes you to use what they refer to as a [managed bridge network](https://linuxcontainers.org/lxd/docs/master/reference/network_bridge/).  In this setup, when you launch a container, LXD will create a [bridge interface](https://wiki.archlinux.org/title/network_bridge) for you automatically for networking in your container.  It then runs `dnsmasq` on your host to do things like DNS and DHCP inside your containers.

While these managed bridge networks are convenient, I found that `dnsmasq` would frequently crash.  I would then lose networking in my container until I restarted LXD.  I realized that I didn't need anything `dnsqmasq` was providing, so I instead setup LXD to use an _unmanaged bridge network_.

An unmanaged bridge network is where you setup a bridge interface on your own, and just hand it to LXD to use.  In this setup, LXD doesn't run `dnsmasq`, so you are free to setup DNS/networking between your host and container however you like.  I found this quite difficult to figure out, so I wanted to put together a guide for anyone else interested.

This guide walks through installing and setting up LXD on the host, creating the LXD NixOS image, and running the LXD NixOS container.

## Setting up the LXD host

The following NixOS module will install LXD and setup a bridge interface that we will use.  It also sets up things like firewall rules and a NAT for accessing the internet from our container.  Comments are inline:

```nix
# This module enables LXD.
#
# This sets up networking for an unmanaged bridge to be used with LXD.
#
# Note that by default LXD uses a managed bridge, that also runs dnsmasq to do
# things like DNS and DHCP to your containers.  I don't need all of that, and
# dnsmasq seems to crash quite often, so this module just sets up an unmanaged
# bridge.

{ config, lib, pkgs, ...}:

{
  # Enable LXD.
  virtualisation.lxd = {
    enable = true;

    # This turns on a few sysctl settings that the LXD documentation recommends
    # for running in production.
    recommendedSysctlSettings = true;
  };

  # This enables lxcfs, which is a FUSE fs that sets up some things so that
  # things like /proc and cgroups work better in lxd containers.
  # See https://linuxcontainers.org/lxcfs/introduction/ for more info.
  #
  # Also note that the lxcfs NixOS option says that in order to make use of
  # lxcfs in the container, you need to include the following NixOS setting
  # in the NixOS container guest configuration:
  #
  # virtualisation.lxc.defaultConfig = "lxc.include = ''${pkgs.lxcfs}/share/lxc/config/common.conf.d/00-lxcfs.conf";
  virtualisation.lxc.lxcfs.enable = true;

  # This sets up a bridge called "mylxdbr0".  This is used to provide NAT'd
  # internet to the guest.  This bridge is manipulated directly by lxd, so we
  # don't need to specify any bridged interfaces here.
  networking.bridges = { mylxdbr0.interfaces = []; };

  # Add an IP address to the bridge interface.
  networking.localCommands = ''
    ip address add 192.168.57.1/24 dev mylxdbr0
  '';

  # Firewall commands allowing traffic to go in and out of the bridge interface
  # (and to the guest LXD instance).  Also sets up the actual NAT masquerade rule.
  networking.firewall.extraCommands = ''
    iptables -A INPUT -i mylxdbr0 -m comment --comment "my rule for LXD network mylxdbr0" -j ACCEPT

    # These three technically aren't needed, since by default the FORWARD and
    # OUTPUT firewalls accept everything everything, but lets keep them in just
    # in case.
    iptables -A FORWARD -o mylxdbr0 -m comment --comment "my rule for LXD network mylxdbr0" -j ACCEPT
    iptables -A FORWARD -i mylxdbr0 -m comment --comment "my rule for LXD network mylxdbr0" -j ACCEPT
    iptables -A OUTPUT -o mylxdbr0 -m comment --comment "my rule for LXD network mylxdbr0" -j ACCEPT

    iptables -t nat -A POSTROUTING -s 192.168.57.0/24 ! -d 192.168.57.0/24 -m comment --comment "my rule for LXD network mylxdbr0" -j MASQUERADE
  '';

  # ip forwarding is needed for NAT'ing to work.
  boot.kernel.sysctl = {
    "net.ipv4.conf.all.forwarding" = true;
    "net.ipv4.conf.default.forwarding" = true;
  };

  # kernel module for forwarding to work
  boot.kernelModules = [ "nf_nat_ftp" ];
}
```

The big take-away from this is that LXD is installed, and we have a bridge interface called `mylxdbr0` that we can use.

I ran this on both nixos-22.05 at commit `c06d5fa9c60`, and nixos-unstable at commit `2da64a81275b68`.  Both of these commits are from around 2022-09-09.

## Setting up LXD

One of the unfortunate things about LXD is that it requires some manual setup.  Unlike most other things in NixOS, LXD is not fully declarative.

Before running `lxc` (the command to interact with the LXD daemon) for the first time, you need to initialize it and setup the default container settings.  You can do this interactively with the command `lxc init`, or you could do this semi-declaratively by passing `lxc init` a "preseed" file with all the settings we want to use:

```console
$ cat my-preseed-file.yaml
config:
  images.auto_update_interval: "0"
networks: {}
storage_pools:
- config:
    source: /var/lib/lxd/storage-pools/default
  description: ""
  name: default
  driver: dir
profiles:
- config: {}
  description: Default LXD profile
  devices:
    root:
      path: /
      pool: default
      type: disk
  name: default
projects:
- config:
    features.images: "true"
    features.networks: "true"
    features.profiles: "true"
    features.storage.volumes: "true"
  description: Default LXD project
  name: default
```

Then tell `lxc` to use this:

```console
$ lxd init --preseed < my-preseed-file.yaml
```

The things to note about this preseed file:

- It sets up a `root` device that just uses a file on disk.  This is simple, but you might want to explicitly run `lxc init` if you want to setup something like a ZFS-backed root filesystem.
- This does not setup a `network`.  We'll explicitly add our unmanaged bridge interface in a later step.

Now we need to create the guest LXD NixOS image.

## Create NixOS image for use as LXD guest

The [`nixos-generators`](https://github.com/nix-community/nixos-generators) tool makes it easy to create a NixOS image for LXD.

First, you need a `configuration.nix` for the NixOS guest in your current directory.  Here's the `configuration.nix` I'm using.  Most things are commented in-line:

```nix
{ config, pkgs, lib, modulesPath, ... }:

{
  imports =
    [ # Need to load some defaults for running in an lxc container.
      # This is explained in:
      # https://github.com/nix-community/nixos-generators/issues/79
      "${modulesPath}/virtualisation/lxc-container.nix"

      # other modules:
      ...
    ];

  # This doesn't do _everything_ we need, because `boot.isContainer` is
  # specifically talking about light-weight NixOS containers, not LXC. But it
  # does at least gives us something to start with.
  boot.isContainer = true;

  # These are the locales that we want to enable.
  i18n.supportedLocales = [ "C.UTF-8/UTF-8" "en_US.UTF-8/UTF-8" "ja_JP.UTF-8/UTF-8" ];

  # Make sure Xlibs are enabled like normal.  This is disabled by
  # lxc-container.nix in imports.
  environment.noXlibs = false;

  # Make sure command-not-found is enabled.  This is disabled by
  # lxc-container.nix in imports.
  programs.command-not-found.enable = true;

  # Disable nixos documentation because it is annoying to build.
  documentation.nixos.enable = false;

  # Make sure documentation for NixOS programs are installed.
  # This is disabled by lxc-container.nix in imports.
  documentation.enable = true;

  # `boot.isContainer` implies NIX_REMOTE = "daemon"
  # (with the comment "Use the host's nix-daemon")
  # We don't want to use the host's nix-daemon.
  environment.variables.NIX_REMOTE = lib.mkForce "";

  # Suppress daemons which will vomit to the log about their unhappiness
  systemd.services."console-getty".enable = false;
  systemd.services."getty@".enable = false;

  # Use flakes
  nix = {
    package = pkgs.nixUnstable;
    extraOptions = ''
      experimental-features = nix-command flakes
    '';
   };

  # We assume that LXD will create this eth1 interface for us.  But we don't
  # use DHCP, so we configure it statically.
  networking.interfaces.eth1.ipv4.addresses = [{
    address = "192.168.57.50";
    prefixLength = 24;
  }];

  # We can access the internet through this interface.
  networking.defaultGateway = {
    address = "192.168.57.1";
    interface = "eth1";
  };

  # The eth1 interface in this container can only be accessed from my laptop
  # (the host).  Unless the host in compromised, I should be able to trust all
  # traffic coming over this interface.
  networking.firewall.trustedInterfaces = [
    "eth1"
  ];

  # Since we don't use DHCP, we need to set our own nameservers.
  networking.nameservers = [ "8.8.4.4" "8.8.8.8" ];

  networking.hostName = "lxc-nixos";

  # This value determines the NixOS release with which your system is to be
  # compatible, in order to avoid breaking some software such as database
  # servers. You should change this only after NixOS release notes say you
  # should.
  system.stateVersion = "22.05"; # Did you read the comment?
}
```

There shouldn't be anything too surprising in here.  You may want to add some other modules to `imports` if you want to install extra programs or services.

Now that you have this `configuration.nix`, you can use `nixos-generators` to create the LXD image:

```console
$ nix-shell -p nixos-generators
$ METAIMG="$(nixos-generate -f lxc-metadata)"
$ IMG="$(nixos-generate -c ./configuration.nix -f lxc)"
```

You can now import this image into LXD.  The image is named `nixos`:

```console
$ lxc image import --alias nixos "${METAIMG}" "${IMG}"
```

Show the image:

```console
$ lxc image show
```

Next we need to create a container based on this image.

## Create the NixOS container

Now we create a container based on this image.  The container is named `lxc-nixos`:

```console
$ lxc init nixos lxc-nixos -c security.nesting=true
```

`-c security.nesting=true` is necessary for using Nix's sandbox in the container.  You probably want to enable this if you intend to build with Nix in the container.

You must now add the unmanaged bridge interface on the host to the container:

```console
$ lxc config device add lxc-nixos eth1 nic nictype=bridged parent=mylxdbr0
```

This command adds a device called `eth1` to the instance `lxc-nixos` where the host interface is called `mylxdbr0`.  The interface in the container will also get called `eth1`.

## Run the NixOS container

You can now finally run the container:

```console
$ lxc start lxc-nixos
```

You can use the following command to confirm the container is running, and confirm the IP address was set correctly in the guest NixOS configuration:

```console
$ lxc list
```

You can start a shell in the container to play around:

```console
$ lxc exec lxc-nixos -- /run/current-system/sw/bin/bash
```

From here, I generally setup the container so I can SSH into it.  I then access the container with `ssh` from the host:

```console
$ ssh me@192.168.57.50
```

You can stop the container by either running `sudo poweroff` from within the container, or from the host:

```console
$ lxc stop lxc-nixos
```

## Conclusion

Following these steps should set you up with an LXD NixOS container.  Setting up a container with an unmanaged bridge is a little bit more work than just using a managed bridge, but a little more robust since you don't need to have `dnsmasq` running on the host.

## Additional Documentation

Here are a few links you might find interesting:

-   <https://srid.ca/lxc-nixos>

    This is an explanation of how to create and use an LXC image on NixOS.  Many of the above steps are originally based on this explanation.

-   [`nixpkgs/nixos/tests/lxd.nix`](https://github.com/NixOS/nixpkgs/blob/6b29af2b84aea67176da8ccb949555e705a04550/nixos/tests/lxd.nix)

    NixOS test for creating LXD image and using it.  This also performs at least some of the above steps.  Since it is a NixOS test, you can be fairly certain it works.

-   <https://discuss.linuxcontainers.org/t/error-starting-instance-with-unmanaged-bridge-about-ovs-vsctl-not-being-found/15068>

    This is a question I asked on the LXD Discourse about how to create an unmanaged bridge.  I had a bunch of trouble figuring this out (but it was mostly because I don't know anything about bridges).

-   <https://discourse.nixos.org/t/example-config-for-nixos-as-host-for-lxd-lxc-containers/14322/2>

    This is an explanation of how to setup LXD declaratively on NixOS.  This might help someone who isn't satisfied with running `lxd init` themselves.

-->
