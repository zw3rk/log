+++
title = "A Haskell Cross Compiler for Raspberry Pi"
date = 2017-05-16T00:00:00+08:00
draft = false
author = "Moritz"
+++

After
[setting
up our Raspberry Pi](https://medium.com/@zw3rk/quick-headless-raspberry-pi-setup-52ad6dd312c4) and
[building
the raspbian SDK](https://medium.com/@zw3rk/raspbian-cross-compilation-toolchain-830fe56d75ba). Today we will build a Haskell cross compiler (GHC)
for the [Raspberry Pi](http://amzn.to/2pykYYv), to be used together
with the SDK.

We will use a custom GHC branch, as not all necessary patches have yet
landed yet in the upstream repository. Alternatively you can use the
official GHC `master` branch and apply the patches manually with
`arc patch`.


## Prerequisites {#prerequisites}

Make sure you have a `ghc` and `cabal` installed. You can download a
recent version from
[downloads.haskell.org](http://downloads.haskell.org/~ghc/8.0.2/). You
will also need `alex` and `happy`:

```text
cabal install alex happy
```

This should produce `~/.cabal/bin/alex` and `~/.cabal/bin/happy`.

Due to an incompatibility between the _latest version_ of libffi (_from
2014_), and recent llvm versions, which only understand the unified arm
assembly syntax, we need to build a custom libffi version. With the
raspbian SDK's `prebuilt/bin` in `PATH` this should be as simple as:

```text
git clone https://github.com/libffi/libffi.git
```

```text
cd libffi
```

```text
./autogen.sh
```

```text
CC="arm-linux-gnueabihf-clang" CXX="arm-linux-gnueabihf-clang" \
        ./configure \
        --prefix=/path/to/libffi/arm-linux-gnueabihf \
        --host=arm-linux-gnueabihf \
        --enable-static=yes --enable-shared=yes
```

```text
make && make install
```

This will build and place the `libffi` header and libraries into
`/path/to/libffi/arm-linux-gnueabihf`.


## Building GHC {#building-ghc}

Ensure that `ghc`, `alex`, `happy`, `cabal`, as well as your Raspbian
SDK are in your `PATH`. If not, export `PATH` appropriately:

```text
export PATH=$HOME/.cabal/bin:$PATH
export PATH=/path/to/bin/ghc:$PATH
export PATH=/path/to/raspbian-sdk/prebuilt/bin:$PATH
```

Next, we will obtain the patched GHC source via `git`:

```text
git clone --recursive git://git.haskell.org/ghc.git
cd ghc
git remote add zw3rk https://github.com/zw3rk/ghc.git
git fetch zw3rk
git checkout zw3rk/my-ghc -b my-ghc
git submodule update --init --recursive
```

_Note: the custom_ `my-ghc` _branch contains a few patches
(_[_D3352_](https://phabricator.haskell.org/D3352)/,/
[_D3443_](https://phabricator.haskell.org/D3443), _/
[_D3448_](https://phabricator.haskell.org/D3448)_,/
[_D3502_](https://phabricator.haskell.org/D3502)/,/
[_D3579_](https://phabricator.haskell.org/D3579)/,/
[_D3591_](https://phabricator.haskell.org/D3591)/) on top of the GHC
master branch that are not yet landed./

Building `GHC` for `arm-linux-gnueabihf` and _installing it alongside_
`clang` _in the raspbian sdk_ should require only the following now

```text
# set paths
export RASPBIAN_SDK=/path/to/raspbian-sdk/
export LIBFFI=/path/to/libffi/arm-linux-gnueabihf
```

```text
# Boot up the build system
./boot
```

```text
# Configure a GHC that targets the Raspberry Pi
./configure --target=arm-linux-gnueabihf \
            --prefix=$RASPBIAN_SDK/prebuilt \
            --with-system-libffi \
            --with-ffi-includes=$LIBFFI/include \
            --with-ffi-libraries=$LIBFFI/lib
```

```text
# Create a mk/build.mk and set the BuildFlavour to quick-cross
sed -E "s/^#(BuildFlavour[ ]+= quick-cross)$/\1/" \
    mk/build.mk.sample > mk/build.mk
```

```text
# Compile and install ghc
make -j && make install
```

This will likely take approximately 30--60 minutes depending on your
hardware. If everything went right, you will find
`arm-linux-gnueabihf-ghc` in `/path/to/raspbian-sdk/prebuilt/bin`.

We will wrap up by trying out our brand new Haskell cross compiler.


## Compiling `Hello World` {#compiling-hello-world}

Let us create a very simple `Hello.hs` similar to the `hello.c` we
created when trying out `clang` from the SDK's toolchain.

```text
module Main where
```

```text
main :: IO ()
main = putStrLn "Hello World!"
```

Compiling it should be as trivial as saying

```text
arm-linux-gnueabihf-ghc Hello.hs
```

Assuming that `/path/to/raspbian-sdk/prebuilt/bin` is still in your
`PATH`.


## Running Hello World {#running-hello-world}

All that is left now, is to copy the binary over to the Raspberry Pi

```text
scp Hello pi@raspberrypi:
```

and executing it

```text
ssh pi@raspberrypi ./Hello
```

Next we will look at Haskell's package management with Cabal with an eye
on cross compilation.