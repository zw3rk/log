+++
title = "What is New in Cross Compiling Haskell"
date = 2017-11-11T00:00:00+08:00
draft = false
author = "Moritz"
+++

## October Edition {#october-edition}

After the announcement and release of experimental cross compiler
previews as binary distributions
[last
month](https://medium.com/@zw3rk/what-is-new-in-cross-compiling-haskell-976cd4752bb9), it turned out that binary distributions produced by the make
base build system were not what I had hoped for.

[GHC 8.4.1](https://ghc.haskell.org/trac/ghc/wiki/Status/GHC-8.4.1) is
around the corner (Release in February 2018. Cut release branch in
November 2017.) and will come with a new
[shake](http://shakebuild.com) based build system called
[hadrian](http://github.com/snowleopard/hadrian), which has been in
development for quite some time. The plan --- I believe --- is that the
make based and the shake based build system will both be with us until
hopefully the make base one will eventually be dropped.

After investigating the current make based build system trying to
produce better cross compiler binary distributions, I bit the bullet and
went ahead with hadrian.

This also lead to the first part of a series of posts on how GHCs build
system works:
[Building
GHC: The package database](https://medium.com/@zw3rk/building-ghc-the-package-database-50c37cf6ce33). More of which I plan to write up soon.

My ideal cross comiler binary distrubion is an compressed archive (zip,
tar.xz) that contains a `bin` and `lib` folder. Simply unpacking it and
running `bin/ghc` should be enough (or in the case of a cross compiler
`bin/<arch>-<vendor>-<os>-ghc`). This however requires that the
distribution is relocatable. While we build relocatable builds for
Windows, we do not for other platforms.

To achieve this, it would be great if the build system would place the
package database relocatable into `lib` and the binaries into `bin`. All
that would be needed to build the binary distribution then would be to
archive and compress those two folders.

To that extend I've spent some time patching GHC, Cabal and hadrian to
make this possible. This has lead to a rather
[large hadrian fork](https://github.com/snowleopard/hadrian/pull/445).

I hope this will provide the foundation to provide better binary
distributions for the cross compilers. As such stay tuned for new binary
distribution releases and hopefully me getting back to actually cross
compilation stuff rather than build system logic.