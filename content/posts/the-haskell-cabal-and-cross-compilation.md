+++
title = "The Haskell Cabal and Cross Compilation"
date = 2017-05-17T00:00:00+08:00
draft = false
author = "Moritz"
+++

Once you want to tap into the vast ecosystem of haskell libraries, you
will run into cabal and hackage in one way or the other _(stack and
stackage build upon cabal as well)_. Over the last few days we
[set
up](https://medium.com/@zw3rk/quick-headless-raspberry-pi-setup-52ad6dd312c4) the [Raspberry Pi](http://amzn.to/2roKzUZ), built the
[Raspbian
SDK](https://medium.com/@zw3rk/making-a-raspbian-cross-compilation-sdk-830fe56d75ba) and the
[Haskell
cross compiler](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-raspberry-pi-ddd9d41ced94). Today we will look at what cabal is, and how to use it
for cross compilation.


## Cabal, cabal-install, cabal and Hackage {#cabal-cabal-install-cabal-and-hackage}

The \*C\*ommon \*A\*rchitecture for \*B\*uilding \*A\*pplications and
\*L\*ibraries consists of [_Cabal_](https://www.haskell.org/cabal/) the
library. _Cabal_ the library contains the logic how to build a haskell
package from a =.cabal= file. _cabal-install_ is the cabal package, that
provides the `cabal` command.
[_Hackage_](https://hackage.haskell.org/) finally is the haskell
package repository.

As an end user you will mostly deal with `cabal` the command line
interface. This will also take care of downloading dependencies and
building them as required.


## Cross Compiling with =cabal= {#cross-compiling-with-cabal}

cabal is not yet very cross compilation agnostic. As cross compilation
has been mostly a niche, cross compiling packages with cabal needs some
hand-holding.

By default `cabal` will use a non-prefixed toolchain, which results in
the library being compiled for the _build_ machine. In the cross
compilation setting, we want to compile for the _host_ machine. That is
the machine that will _host_ the final binary, and not the machine that
_builds_ the binary.

Luckily `cabal` provides the necessary arguments to pass in the
toolchain we want to use _(I've seen this first in the_
[_ghc-ios-scripts_](https://github.com/ghc-ios/ghc-ios-scripts)/)/.

For our `arm-linux-gnueabihf-ghc` we
[built
yesterday](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-raspberry-pi-ddd9d41ced94) we want to effectively call `cabal` with the following
arguments:

```text
--builddir=dist/arm-linux-gnueabihf
```

to put any build product into a separate directory for each
architecture. And

```text
--with-ghc=arm-linux-gnueabihf-ghc
--with-ghc-pkg=arm-linux-gnueabihf-ghc-pkg
--with-gcc=arm-linux-gnueabihf-clang
--with-ld=arm-linux-gnueabihf-ld
```

to teach cabal about our toolchain. cabal may call `hsc2hs` as well,
passing

```text
--hsc2hs-options=--cross-compile
```

when compiling ensures that `hsc2hs`
[operates
in cross compilation mode](http://downloads.haskell.org/~ghc/master/users-guide/utils.html#hsc2hs-cross).

Finally `cabal` may invoke `configure`. Therefore we pass

```text
--configure-option=--host=arm-linux-gnueabihf
```

as well, when running `cabal configure` or `cabal install`, so that
configure scripts can rely on the proper `--host` flag.

Passing all this by hand is truly annoying. Therefore let us create a
wrapper script for this _(again credit for this goes to_
[_ghc-ios-scripts_](https://github.com/ghc-ios/ghc-ios-scripts)/,
where I saw this approach first --- we will use a slightly modified
version here)./

Let us create a script called `cabal-wrapper` with the following
content:

```text
#!/bin/bash

name=${0##*/}
cmd=${name##*-}
target=${name%-*}

fcommon="--builddir=dist/${target}"
fcompile=" --with-ghc=${target}-ghc"
fcompile+=" --with-ghc-pkg=${target}-ghc-pkg"
fcompile+=" --with-gcc=${target}-clang"
fcompile+=" --with-ld=${target}-ld"
fcompile+=" --hsc2hs-options=--cross-compile"
fconfig="--disable-shared --configure-option=--host=${target}"

case $1 in
  configure|install) flags="${fcommon} ${fcompile} ${fconfig}" ;;
  build)             flags="${fcommon} ${fcompile}" ;;
  list|info|update)  flags="" ;;
  "")                flags="" ;;
  *)                 flags=$fcommon ;;
esac

exec $cmd $flags "$@"
```

_Note: for simplicity we do not wrap_ `cabal`'s new
[_nix-style
local build commands_](http://blog.ezyang.com/2016/05/announcing-cabal-new-build-nix-style-local-builds/) _like_ `new-build`, and `new-install`.

Simply creating a symbolic link

```text
ln -s cabal-wrapper arm-linux-gnueabihf-cabal
```

provides us with `arm-linux-gnueabihf-cabal` which nicely fits in with
the rest of the toolchain we built so far. And we can use
`arm-linux-gnueabihf-cabal` just like `cabal` to build or install
haskell packages to use with the cross compiler.


## Be Aware of build-type: Custom {#be-aware-of-build-type-custom}

Cabal provides an escape hatch for to support custom build types, for
when the `build-type: Simple` is not sufficient. Unfortunately this
requires the `Setup.hs` to be built by cabal and run, and cabal default
to using the `ghc` is knows about to compile the `Setup.hs`. This then
leads to a `Setup.hs` that is compiled with the cross compiling ghc
(here: `arm-linux-gnueabihf-ghc`), and can only be run on the _host_,
but not on the _build_ machine. Ideally cabal would be cross compilation
aware and compile the `Setup.hs` with the compiler that targets the
_build_ machine.

As quite a few packages use `build-type: Custom` for the purpose of
supporting `doctest` or `haddock`. Often times just rewriting
`build-type: Custom` to `build-type: Simple` can make a package succeed
to compile for cross compilation. _(An alternative solution, that tries
to respect the_ =build-type: Custom=/, can be found in the/
[_cabal-custom_](https://github.com/ghc-ios/ghc-ios-scripts/blob/master/i386-apple-darwin11-cabal-custom)
_script, again from the_
[_ghc-ios-scripts_](https://github.com/ghc-ios/ghc-ios-scripts)/)/


## Be Aware of Template Haskell {#be-aware-of-template-haskell}

Template Haskell provides interesting challenges for cross compilation.
For a long time Template Haskell was simply not an option for cross
compilation. While there are
[evil
splicer](https://git-annex.branchable.com/design/assistant/blog/day%5F236%5F%5Fevil%5Fsplicer/) and [ZeroTH](https://hackage.haskell.org/package/zeroth),
which try to work around Template Haskell, there has been no proper
Template Haskell support. However
[GHCJS](https://github.com/ghcjs/ghcjs) does support Template Haskell
and is a cross compiler as well, and the same approach can be taken with
GHC. Cross compiling and Template Haskell will the the main topic for
next week.