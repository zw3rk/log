+++
title = "Relocatable GHC Cross Compiler Binary Distributions"
date = 2017-12-20T00:00:00+08:00
draft = false
author = "Moritz"
+++

> \*/TLDR/\*/: Watch the videos at the end of this article and grab your
> cross compilers from/
> [_<http://hackage.mobilehaskell.org>_](http://hackage.mobilehaskell.org/)

It's been a while since the
[GHC
Cross Compiler Binary Distributions](https://medium.com/@zw3rk/ghc-cross-compiler-binary-distributions-490bb2c0c411) post. A primary issue was
installation. Specifically the =./configure= and `make install` logic
that is part of GHCs binary distributions. This is necessary for GHC to
install properly and setup the `settings` file, which contains system
specific information that can only be detected on the final host. These
include for example a check to see if the host does position independent
executables (PIE) by default or not; and if not detected properly will
result in a compiler that can not produce binaries for the host.

For cross compilers this is a little different, as they are tightly
coupled to the cross compilation tool chain. As such there is less
freedom in how the final host will behave and a lot more knowledge about
the hosts tool chain and the final target are known at build time and
many of the checks performed in the =./configure= script are not suited
for the installation of a cross compiler.

To that end, as I've mentioned in
[What
is New in Cross Compiling Haskell: October Edition](https://medium.com/@zw3rk/what-is-new-in-cross-compiling-haskell-b70decd4b3a6) and
[What
is New in Cross Compiling Haskell: November Edition](https://medium.com/@zw3rk/what-is-new-in-cross-compiling-haskell-56b9385ed93), I've put a lot of
effort into allowing the [shake](http://shakebuild.com/) base
[GHC build system](https://github.com/snowleopard/hadrian/), and cabal
to produce relocatable binary distributions of GHC.

What do I mean by relocatable binary distributions? Essentially that you
can just grab the tar or zip archive of GHC, unpack it and execute
`bin/ghc` with no need to actually run =./configure= and `make install`
if your system is similar enough to the build system (e.g. the usual
case for cross compilers with a deterministic cross compilation tool
chain). If you still need to adapt GHC to your host, you can
run =./configure= right inside the unpacked folder prior to executing
`bin/ghc`. [Alp Mestanogullari](https://github.com/alpmestan) has done
a fantastic job in bringing the =./configure= logic to the relocatable
binary distributions and is working on getting tests and documentation
working with relocatable binary distributions as well!

One of the major benefits of the relocatable binary distributions is
that when GHC is built, it builds into `_build/stage<N>/bin` and
`_build/stage<N>/lib` and those folders can be freely moved around for
`stage1` and later on macOS, linux, windows. Why only on those three
systems? This has primarily to do with how GHC figures out the location
of the executable that is invoked, as it needs to know where its `lib`
folder is. This turns out to be a rather tricky to get right and is
rather platform dependent. /(I have not tried \*BSD; but I believe that
could work as well, if someone wants to give this a try, let me
know! --- If you want to add support for any other platform, the best
point to start would be to make/ `getExecutablePath` _from_
`base:System.Environment` _behave correctly for your platform.)_

Another benefit is that we do not necessarily need the wrapper scripts
anymore. The wrappers used to wrap the binaries and add the `-B` flag or
similar to tell the binaries where the `lib` folder was. As such you can
just run `gdb ghc` or `lldb ghc`. _(The wrapper scripts are still
generated when installing ghc via_ `make install` _if the_ `lib` _and_
`bin` _folders are /=./configure=/d to be in different locations.)_

This work is by no means concluded yet but it is sufficiently advanced
that I was able to produce a set of relocatable cross compilers for iOS,
Android and Raspberry Pi, which you can find at
[http://hackage.mobilehaskell.org](http://hackage.mobilehaskell.org/).
If you run into issues, please file them with the
[mobilehaskell
hackage-overlay issue tracker](https://github.com/mobilehaskell/hackage-overlay/issues).

> A word of caution though: as `cabal=s =new-build` has a rather
> [annoying bug](https://github.com/haskell/cabal/issues/4939) with
> respect to forwarding flags to dependencies, my advice is to unpack or
> add submodules for dependencies that require cabal to run `hsc2hs`
> or =./configure=.

<!--quoteend-->

> I plan to get around to fixing this; but I haven't manage to do so
> just yet.

I've use the aforementioned binary distributions to build a stitching
application where the logic part (stitching together simple screenshots
into a long continuous screenshot) is written in Haskell and the UI is
written in kotlin for Android and obj-c (make calling haskell via c a
bit easier than swift) for iOS, which you see below.

Stitcher running on an Asus Zenfone
Stitcher running on an iPhone 7
I plan to write up building Stitcher as a series starting in 2018, and
the accompanying code will be available on GitHub.

---

_If you like what you read and want to support further work like this,
feel free to support me via_
[_patreon_](https://www.patreon.com/zw3rk) _or send cryptocurrencies
to_

---

<a id="orgb1a27e9"></a>
[**Mobile Haskell (@mobilehaskell) |
Twitter**\\\\
/The latest Tweets from Mobile Haskell (@mobilehaskell). Advances in
using Haskell for mobile development Tweets
by.../twitter.com](https://twitter.com/@mobilehaskell)[[<https://twitter.com/@mobilehaskell>][]]

<a id="orgaff58fc"></a>
[**zw3rk (@zw3rktech) | Twitter**\\\\
/The latest Tweets from zw3rk (@zw3rktech). your friendly haskell
consultants/twitter.com](https://twitter.com/@zw3rktech)[[<https://twitter.com/@zw3rktech>][]]

<a id="org8b09621"></a>
[**Moritz Angermann (@angerman\_io) |
Twitter**\\\\
/The latest Tweets from Moritz Angermann (@angerman\_io). Type masseur.
Singapore/twitter.com](https://twitter.com/@angerman%5Fio)[[<https://twitter.com/@angerman%5Fio>][]]