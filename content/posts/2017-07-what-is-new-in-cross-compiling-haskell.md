+++
title = "What is New in Cross Compiling Haskell"
date = 2017-08-03T00:00:00+08:00
draft = false
author = "Moritz"
+++

## July Edition {#july-edition}

Sadly I wasn't able to write anything throughout July. I hope I will be
able to carve out some time this month to write about the new features
we have been working on.

The focus has mainly been on making the compilation experience for cross
compilers as pleasant as possible and figuring out how to build binary
distributions of GHC cross compilers.

One of the rather annoying kludges has been the need for the
`libtool-lite` script on some platforms; this should be unnecessary
soon. And while we still need the wrapper file for cabal,
[cabal has learned about](https://github.com/haskell/cabal/pull/4617)
`-staticlib`, and `new-build` can now be used to produce a static
library of the project (with GHCs that support `-staticlib`).

I hope you are excited for the posts to come! As always you can follow
us here on medium, on [github](https://github.com/zw3rk), or on
[twitter](http://twitter.com/zw3rktech).