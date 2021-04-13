+++
title = "Building iconv for Android"
date = 2017-05-03T00:00:00+08:00
draft = false
author = "Moritz"
+++

iconv (**i\*nternational \*con\*version) is provided by `libiconv`, and
available through \*GHC.IO.Encoding** in the **base** package.

As this is an external dependency, it needs to be available for a build
of GHC to succeed. In the case of cross compiling GHC for iOS, we can
use the `libiconv.dylib` provided by the iOS sdk. For android however,
we'll need to build `libiconv.so`.

Building `libiconv` version `1.15` for `aarch64/android` on `macOS` via
androids ndk toolchain can be done as follows:

```text
curl -O -L https://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.15.tar.gz
tar xzf libiconv-1.15.tar.gz
cd libiconv-1.15
export ANDROID_HOST=aarch64-linux-android
export ANDROID_BUILD=darwin-x86_64
export ANDROID_ARCH=arm64
export ANDROID_NDK=$HOME/Library/Android/sdk/ndk-bundle
export ANDROID_VERSION=24
export ANDROID_TOOLCHAIN_VERSION=4.9
export ANDROID_SYSROOT=$ANDROID_NDK/platforms/android-$ANDROID_VERSION/arch-$ANDROID_ARCH
export CFLAGS=--sysroot=$ANDROID_SYSROOT
export CPPFLAGS=--sysroot=$ANDROID_SYSROOT
export AR=$ANDROID_HOST-ar
export RANLIB=$ANDROID_HOST-ranlib
export PATH=$ANDROID_NDK/toolchains/$ANDROID_HOST-$ANDROID_TOOLCHAIN_VERSION/prebuilt/$ANDROID_BUILD/bin:$PATH
./configure --host=$ANDROID_HOST --with-sysroot=$ANDROID_SYSROOT
make
```

Building on linux should be similar.

A slightly more elaborate script to build `libiconv` for `armv7` and
`aarch64` can be found in the `zw3rk/ghc-build-scripts` repository as
`build-libiconv`. The script reduces downloading and build `libiconv`
for android to:

```text
./build-libiconv --host=darwin-x86_64 --prefix=libiconv-prefix
```

and places the build products into
`libiconv-prefix/{aarch64-linux-android,arm-linux-androideabi}`.

_Note: The_ `build-libiconv` _script builds the static library_
`libiconv.a` _which can be linked into the binary. If you want the
shared object, change the line that reads_
`--enable-shared=no --enable-static=yes` _into_
=--enable-shared=yes --enable-static=yes=/./