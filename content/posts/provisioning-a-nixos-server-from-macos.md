+++
title = "Provisioning a NixOS server from macOS"
date = 2018-01-17T00:00:00+08:00
draft = false
author = "Moritz"
+++

[Nix](https://nixos.org/nix/) is a _purely functional package manager_
and [NixOS](https://nixos.org/) is a _purely functional Linux
distribution._ I have been running a NixOS server for a while now, and
been quite happy with the configuration. However updating it has
essentially been copying the `configuration.nix` file over and calling
`nixos-rebuild switch --upgrade`. This builds packages that are not in
the cache on the server. [NixOps](https://nixos.org/nixops/) is _the
NixOS cloud deployment tool_ and I should be able to use it to provision
the server. Specifically using NixOps on macOS seems to be rather
uncommon. I'll detail my process, so that we all may benefit from using
NixOps on macOS to provision a NixOS linux server.


## Motivation {#motivation}

Over the last few days I've moved
[hackage.mobilehaskell.org](http://hackage.mobilehaskell.org) over to
S3 and fastly. I've also started adding automation, so that changes
(patches to cabal packages) in the
[hackage-overlay](http://github.com/mobilehaskell/hackage-overlay)
repository will automatically be reflected on
[hackage.mobilehaskell.org](http://hackage.mobilehaskell.org). In
general I prefer my software to not be built on the server, but rather
on some build machine, and then only have the final binary be deployed
on the server. NixOps with a build slave allows for this setup.


## Setting up Nix and Nix on Darwin (macOS) {#setting-up-nix-and-nix-on-darwin--macos}

_Note: when ever running scripts from the internet, verify that the code
you run is what you actually want to run!_

Setting up `nix` and `nix-darwin` has become quite easy these days.
Installing `nix` can be done with

```text
$ curl https://nixos.org/nix/install | sh
```

and installing `nix-darwin` is similarly only a shell command away

```text
$ bash <(curl https://raw.githubusercontent.com/LnL7/nix-darwin/master/bootstrap.sh)
```

If this all installed successfully, you should find `darwin-rebuild` and
`darwin-option` in your `$PATH`.


## Setting up a docker build slave {#setting-up-a-docker-build-slave}

Now that we have nix installed, we still can't build software for our
server, as the server runs `x86_64-linux` but we are on `x86_64-darwin`.
Luckily there is [docker for mac](https://www.docker.com/docker-mac)
which allows us to run Linux contains on macOS, and
[Daiderd Jordan](https://github.com/LnL7) provides not only
`nix-darwin` but also `nix-docker`.

With docker for mac installed we can try out a container

```text
$ docker run --rm -it lnl7/nix nix-repl '<nixpkgs>'
```

which should drop us right into the `nix-repl` in a docker
`x86_64-linux` container, which we can verify with

```text
nix-repl> system
"x86_64-linux"
```

To turn that into a build slave, we'll need to run

```text
$ docker run --restart always --name nix-docker -d -p 3022:22 lnl7/nix:ssh
```

[add
the necessary ssh key](https://github.com/LnL7/nix-docker#running-as-a-remote-builder)

```text
$ git clone https://github.com/LnL7/nix-docker.git
$ mkdir -p /etc/nix
$ chmod 600 ssh/insecure_rsa
$ cp ssh/insecure_rsa /etc/nix/docker_rsa
```

and add the ssh configuration to `~root/.ssh/config`

```text
Host nix-docker
  User root
  HostName 127.0.0.1
  Port 3022
  IdentityFile /etc/nix/docker_rsa
```

Why do we need to add it to `root`? Because the `nix-daemon` will run
the build as root. Now try to connect to the machine as `root`

```text
$ sudo ssh nix-docker
```

and answer yes to add the host to the list of known hosts.


## Teaching nix about the x86\_64-linux build slave {#teaching-nix-about-the-x86-64-linux-build-slave}

At this point we have our build slave working, but nix doesn't know
about it yet. You will need to set

```text
nix.distributedBuilds = true;
nix.buildMachines = [ {
  hostName = "nix-docker";
  sshUser = "root";
  sshKey = "/etc/nix/docker_rsa";
  systems = [ "x86_64-linux" ];
  maxJobs = 2;
} ];
```

in your `~/.nixpkgs/darwin-configuration.nix` to tell nix about your
build slave. You will also need

```text
services.nix-daemon.enable = true;
```

to ensure that the `nix-daemon` is launched with the proper environment
variables (see cat `/Library/LaunchDaemons/org.nixos.nix-daemon.plist`
after a rebuild). And run

```text
$ darwin-rebuild switch
```


## Test: building a package for x86\_64-linux {#test-building-a-package-for-x86-64-linux}

To verify the setup works we will build a slightly modified `hello` for
`x86_64-linux` with `nix-build`

```text
$ nix-build -E 'with import <nixpkgs> { system = "x86_64-linux"; }; hello.overrideAttrs (drv: { rebuild = builtins.currentTime; })'
```

_Note:_ `--check` _won't work, as it will not respect the_ `system`
_setting!_


## NixOps {#nixops}

Using `nixops` at this point should be rather easy. All you need to
ensure is that your configuration contains the
`nixpkgs.system = "x86_64-linux"` attribute.

```text
{ webserver = { config, pkgs, lib, ... }:
  { deployment.targetHost = "...";
    nixpkgs.system = "x86_64-linux";
    ...
  };
}
```

**Note:** with _Nix 2_ this needs to be `nixpkgs.localSystem.system`
instead of `nixpkgs.system` as
[Jezen Thomas](https://medium.com/u/6e2f7c3e0744) kindly noted in the
comments.

With that set, you should be able to run `nixops deploy`

```text
$ nixops deploy -d myserver
building all machine configurations...
...
webserver> copying closure...
myserver> closures copied successfully
webserver> activating the configuration...
webserver> setting up /etc...
...
webserver> activation finished successfully
myserver> deployment finished successfully
```

and have your NixOS server successfully deployed from macOS.

---

_If you like what you read and want to support further work like this,
feel free to support me via_
[_patreon_](https://www.patreon.com/zw3rk) _or send cryptocurrencies
to_

---

<a id="org023ecc0"></a>
[**Mobile Haskell (@mobilehaskell) |
Twitter**\\\\
/The latest Tweets from Mobile Haskell (@mobilehaskell). Advances in
using Haskell for mobile development Tweets
by.../twitter.com](https://twitter.com/@mobilehaskell)[[<https://twitter.com/@mobilehaskell>][]]

<a id="org93c98bb"></a>
[**zw3rk (@zw3rktech) | Twitter**\\\\
/The latest Tweets from zw3rk (@zw3rktech). your friendly haskell
consultants/twitter.com](https://twitter.com/@zw3rktech)[[<https://twitter.com/@zw3rktech>][]]

<a id="org36efa98"></a>
[**Moritz Angermann (@angerman\_io) |
Twitter**\\\\
/The latest Tweets from Moritz Angermann (@angerman\_io). Type masseur.
Singapore/twitter.com](https://twitter.com/@angerman%5Fio)[[<https://twitter.com/@angerman%5Fio>][]]