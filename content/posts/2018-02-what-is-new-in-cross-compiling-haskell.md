+++
title = "What is New in Cross Compiling Haskell"
date = 2018-03-02T00:00:00+08:00
draft = false
author = "Moritz"
+++

## February Edition {#february-edition}

While I didn't manage to find the time to write anything here in
February, I did make some progress on the cross compilation front.

`cabal` can now be invoked outside of the source folder, and tries to
not pollute the source tree with generated files anymore
([haskell/cabal#4874](https://github.com/haskell/cabal/pull/4874)).
The `--with-PROG` flag is now properly respected when using `new-build`;
this allows `new-build` to work with cross compilers
([haskell/cabal#5018](https://github.com/haskell/cabal/pull/5018)).
I've also spent quite some time to get cabals CI back into shape, which
was hitting a rather obscure bug
([haskell/cabal#5132](https://github.com/haskell/cabal/pull/5132)).

I've also started working for [IOHK](https://iohk.io) where I will be
assisting the DevOps team to Cross Compile Haskell with GHC from Linux
to Windows. I'm very excited about the work we do at IOHK, and this will
definitely broaden and improve GHCs cross compilation capabilities.

This does not mean I'll stop working on the mobile cross compilers; I
will still keep improving them --- as I have been --- and I think we are
already on a very good road to get this all sorted in GHC 8.6. While my
initial target was GHC 8.4, we just couldn't get it all into GHC 8.4 in
time.

If everything works out as I hope, GHC 8.6 can be built by default with
the shake based build system: hadrian, and will have extensive cross
compilation capabilities to various platforms.

I'd like to close with some progress we are making at IOHK with respect
to cross compiling Haskell to Windows.

> [[<https://twitter.com/angerman%5Fio/status/969546657141420032>][]]