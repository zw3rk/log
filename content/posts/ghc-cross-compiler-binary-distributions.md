+++
title = "GHC Cross Compiler Binary Distributions"
date = 2017-10-20T00:00:00+08:00
draft = false
author = "Moritz"
+++

As mentioned in the September Edition of
[What
is New in Cross Compiling Haskell](https://medium.com/@zw3rk/what-is-new-in-cross-compiling-haskell-976cd4752bb9), I've been working on producing
binary distributions of cross compilers, so that installing a cross
compiler becomes almost as trivial as installing a GHC binary
distribution.

A word of caution is in order. /These are rather experimental builds,
and realistically I expect there to be a few issues, that will arise.
I'd still appreciate early feedback so that we can iron out the wrinkles
quickly!/

I've build binary distributions for from _macOS Sierra_ to _iOS,
Android_ and _Raspberry Pi_, as well as binary distributions from
_Debian 8_ to _Android_ and _Raspberry Pi_.

So head on over to
[hackage.mobilehaskell.org](http://hackage.mobilehaskell.org) and grab
the binary distribution of your choice.


## Prerequisites {#prerequisites}

For iOS development, the iOS SDK that is shipped with Xcode is required.
For Android you will need the Android NDK, and
[iconv
for Android](https://medium.com/@zw3rk/building-iconv-for-android-e3581a52668f). For Raspberry Pi, you will need to creat the
[Raspbian
SDK](https://medium.com/@zw3rk/making-a-raspbian-cross-compilation-sdk-830fe56d75ba).

You will also need a copy of LLVM5 in `PATH`. As well as a copy of the
bootstrapped toolchain wrapper.

```text
$ git clone https://github.com/zw3rk/toolchain-wrapper.git
$ cd toolchain-wrapper
toolchain-wrapper $ ./bootstrap
$ export PATH="$(PWD)/toolchain-wrapper:$PATH"
```

Note: _you may need to adjust the_ `*-toolchain.config` _file to match
your toolchain paths._


## Installing the binary distribution {#installing-the-binary-distribution}

While I hope that installing the binary distributions will eventually be
as simple as =./configure --prefix=... && make install= for now you will
likely need to provide `--target`... --host=... --build=...= as well.
E.g. for the `aarch64-unknown-linux-android` on Debian 8, you would
provide:

```text
./configure --prefix=... \
            --target=aarch64-linux-android \
            --host=x86_64-pc-linux-gnu \
            --build=x86_64-pc-linux-gnu
make install
```

to satisfy the `configure` script.


## Where to go from here {#where-to-go-from-here}

You should now be able to compile a simple

```text
main = putStrLn "Hello World"
```

haskell file with your `<target>-ghc` for your target. Note that for iOS
and Android you would likely want to use `-staticlib` to produce a
static archive to link with your application.


## Bugs & Issues {#bugs-and-issues}

Please file any bug and issues you run into with the
[hackage-overlay](https://github.com/mobilehaskell/hackage-overlay/issues)
issue tracker. That way we will have a central issue and knowledge base.