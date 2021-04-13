+++
title = "What is New in Cross Compiling Haskell"
date = 2017-10-08T00:00:00+08:00
draft = false
author = "Moritz"
+++

## September Edition {#september-edition}

ICFP was a blast, and I think we made quite some progress in getting
some of the cross compilation diffs merged. Facetime does pay off!

The Q monad extension for template haskell
([D3608](https://phabricator.haskell.org/D3608)) will need some
adjustments going forward. I hope these will be sorted out by the end of
October.

Herbert Valerio Riedel
[announced
the head.hackage hackage overlay](https://mail.haskell.org/pipermail/ghc-devs/2017-September/014682.html). This allows to have a set of patches
which are turned into a separete hackage repository, and the packages
are picked from that repository rather than from the upstream hackage
repository if a patched package exists. This is somewhat similar to how
[Eta](https://eta-lang.org)'s Etlas works. Though as far as I
understand, Etlas does the patching on the client side, whereas the
hackage overlay approach patches the packages on the server side and
provides a separte hackage repository, which takes preceedence over the
upstream hackage repository.

As such I've asked Herbert to give me a jump start on builing hackage
overlays and now
[hackage.mobilehaskell.org](http://hackage.mobilehaskell.org) is live.
It contains so far only a single patched package, namely `zlib`. The
patch works around a cross compilation complication with `hsc2hs`. The
patches can be found in the
[mobilehaskell/hackage-overlay](https://github.com/mobilehaskell/hackage-overlay)
github repository.

[hackage.mobilehaskell.org](http://hackage.mobilehaskell.org) also
contains a highly experimental preview of ghc binary distributions with
iOS, Android and Raspberry Pi cross compilers for macOS Sierra, as well
as Android and Raspberry Pi cross compilers for Linux (debian 8) built
with the `llvmng` llvm backend.

I hope to write some more about the `llvmng` backend, the hackage
overlay, as well as the binary distributions throughout October.