+++
title = "What is New in Cross Compiling Haskell"
date = 2018-08-14T00:00:00+08:00
draft = false
author = "Moritz"
+++

## July Edition {#july-edition}

In July I've been playing around a bit more with `-target`. At this
point I believe the best solution is to have a minimal `ghc` that
doesn't ship with any libraries; and all libraries are built on demand
per target. We will likely want to pre-build and ship the Runtime System
Library `rts` as we do not have a cabal package that would just build
the `rts`. You'd have to specify the targets for which you want to
build/include the `rts`. The drawback is that you'd need the (partial)
target toolchain to build the `rts` for all the bundled =rts=s you want
to ship.

On the other side you'd likely want to use `iserv` (e.g. the
`-fexternal-interpreter`), with which I've just recently run into some
strange behaviour while compiling test-suites for packages, where the
`iserv` process complains about code that is loaded multiple times. I'm
currently exploring how we can get proper test-coverage for libraries,
and maybe even `ghc` in a cross compiled setting.

A few bugs were fixed, the `-staticlib` argument now doesn't fail if the
object files in the archives it's trying to concatenate are of
odd-length; GHC doesn't PANIC anymore when `-jN`, `N>1` is used and it
fails to find/load a library.

I've also updated the relevant llvmng code to work with ghc8.6. I've had
to retract the performance improvement though; as it kept producing
invalid binaries occasionally and I haven't found the reason yet. As
such I'm probably going to rent some compute time on AWS or similar
service to build the cross compiler once the final 8.6.1 hits.