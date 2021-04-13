+++
title = "A Haskell Cross Compiler for Android"
date = 2017-05-30T00:00:00+08:00
draft = false
author = "Moritz"
+++

Over the last two weeks we saw how to build
[a
Haskell Cross Compiler for Raspberry Pi](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-raspberry-pi-ddd9d41ced94), set up
[Cabal
for Cross Compilation](https://medium.com/@zw3rk/the-haskell-cabal-and-cross-compilation-e9885fd5e2f), and how to
[Cross
Compile Template Haskell](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914). Building a Haskell cross compiler for
Android is almost identical, with only minor differences.

For the Raspbian Haskell cross compiler we had a single architecture
only. Android runs on a plethora of architectures. We will focus on arm
processors, specifically the 32bit `armv7` and 64bit `aarch64`.


## The Android NDK & LLVM {#the-android-ndk-and-llvm}

Google
[provides the
NDK for Android](https://developer.android.com/ndk/downloads/index.html), which provides a similar set of tools as the
[Raspbian
Cross Compilation SDK](https://medium.com/@zw3rk/making-a-raspbian-cross-compilation-sdk-830fe56d75ba) does; it contains the Android toolchain and
sysroot.

For GHC we need `opt` and `llc` from the llvm 4, which can be obtained
from their [release download
website](http://releases.llvm.org/download.html).


## Toolchain Wrapping {#toolchain-wrapping}

To keep our `PATH` tidy, and abstract about the Android NDK a bit, we'll
use a wrapper script that wraps the toolchain and embeds the sysroot.

```text
#!/bin/bash
source android-toolchain.config
```

```text
name=${0##*/}
cmd=${name##*-}
target=${name%-*}
```

```text
case $name in
 *-cabal)
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
  ;;
 # android (armv7)
 armv7-linux-androideabi-clang)
  flags=" --target=${target}"
  flags+=" --sysroot=${ADR32_SYSROOT}"
  flags+=" -isysroot ${ADR32_SYSROOT}"
  ;;
 armv7-linux-androideabi-ld|armv7-linux-androideabi-ld.gold)
  flags=" --sysroot=${ADR32_SYSROOT}"
  flags+=" -L${ADR32_TOOLCHAIN_LIB}"
  ;;
 # android (aarch64)
 aarch64-linux-android-clang)
  flags=" --target=${target}"
  flags+=" --sysroot=${ADR64_SYSROOT}"
  flags+=" -isysroot ${ADR64_SYSROOT}"
  ;;
 aarch64-linux-android-ld|aarch64-linux-android-ld.gold)
  flags=" --sysroot=${ADR64_SYSROOT}"
  flags+=" -L${ADR64_TOOLCHAIN_LIB}"
  ;;
 # default
 *-nm|*-ar|*-ranlib) ;;
 *) echo "Unknown command: ${0##*/}" >&2; exit 1;;
esac
```

```text
case $target in
 armv7-linux-android*)
   exec env PATH="${ADR32_PATH}:${PATH}" $cmd $flags "$@" ;;
 aarch64-linux-android*)
   exec env PATH="${ADR64_PATH}:${PATH}" $cmd $flags "$@" ;;
 *) exec $cmd $flags "$@" ;;
esac
```

The `wrapper` depends on `android-toolchain.config` which can be
obtained from the
[zw3rk/toolchain-wrapper](https://github.com/zw3rk/toolchain-wrapper)
repository. _The_ `android-toolchain.config` _will likely need minor
modifications, encoding the location of the NDK._

Next, we will create symbolic links to the wrapper script:

```text
for target in "armv7-linux-androideabi aarch64-linux-android"; do
  for command in "clang ld ld.gold nm ar ranlib cabal"; do
    ln -s wrapper $target-$command
  done
done
```

This will produce 14 files (e.g. `armv7-linux-androideabi-clang`), which
will point to the `wrapper`. The wrapper in turn will build up the
necessary flags to pass to the command, based on the name of the file.
_Note: we assume that_ `ld.bfd` _and_ `ld.gold` _accept the same flags._


## Prerequisites {#prerequisites}

As Android does not ship with iconv by default, and GHC depends on
iconv, we will need to build it as laid out in
[building
iconv for android](https://medium.com/@zw3rk/building-iconv-for-android-e3581a52668f). Note that you do want to build for both targets:
`armv7` and `aarch64` and you want to build **static** libraries (pass
`--enable-shared=no --enable-static=yes` to the `configure` script).
This will ease integrating the library into android studio.

To build GHC, we need `ghc` and `cabal`, as well as `alex` and `happy`.
A recent GHC version from
[downloads.haskell.org](http://downloads.haskell.org/~ghc/8.0.2/)
should provide `ghc` and `cabal`. `alex` and `happy` can then be
installed via `cabal`:

```text
cabal install alex happy
```

As with the
[Haskell
cross compiler for Raspberry Pi](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-raspberry-pi-ddd9d41ced94), we need to build a newer `libffi`
from source, due to an incompatibility between the _latest release
version_ of libffi (_from 2014_), and recent llvm versions. With the
wrapped toolchain in `PATH`, building `libffi` should be as simple as:

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
CC="armv7-linux-androideabi-clang" \
CXX="armv7-linux-androideabi-clang" \
        ./configure \
        --prefix=/path/to/libffi/armv7-linux-androideabi \
        --host=armv7-linux-androideabi \
        --enable-static=yes --enable-shared=yes
```

```text
make && make install
```

```text
git clean -f -x -d
```

```text
./autogen.sh
```

```text
CC="aarch64-linux-android-clang" \
CXX="aarch64-linux-android-clang" \
        ./configure \
        --prefix=/path/to/libffi/aarch64-linux-android \
        --host=aarch64-linux-android \
        --enable-static=yes --enable-shared=yes
```

```text
make && make install
```

This will build and place the `libffi` header and libraries for `armv7`
and `aarch64` into `/path/to/libffi/armv7-linux-androideabi` and
`/path/to/libffi/aarch64-linux-android`.

As we will also be using GHCs `-staticlib` flag. GHC uses `libtool` for
`-staticlib`. As the NDK does not ship `libtool`, we need a thin
wrapper. `libtool-lite` from the
[zw3rk/toolchain-wrapper](https://github.com/zw3rk/toolchain-wrapper)
repository can be used instead; it uses `ar` and `ranlib` under the
hood. We only need to create symbolic links pointing to it:

```text
ln -s libtool-lite armv7-linux-androideabi-libtool
ln -s libtool-lite aarch64-linux-android-libtool
```


## Building GHC {#building-ghc}

We need to build GHC for both targets: `armv7` and `aarch64`. With
`ghc`, `alex`, `happy`, and `cabal` in `PATH`, as well as our wrapped
toolchain:

```text
export PATH=$HOME/.cabal/bin:$PATH
export PATH=/path/to/bin/ghc:$PATH
export PATH=/path/to/wrapped-toolchain:$PATH
```

And a copy of the patched GHC:

```text
git clone --recursive git://git.haskell.org/ghc.git
cd ghc
git remote add zw3rk https://github.com/zw3rk/ghc.git
git fetch zw3rk
git checkout zw3rk/my-ghc -b my-ghc
git submodule update --init --recursive
```

Building GHC for `armv7-linux-androideabi` and `aarch64-linux-android`
should require nothing more than:

```text
# set paths
export PREFIX=/my/prefix
```

```text
export LIBFFI=/path/to/libffi
export LIBICONV=/path/to/libiconv
```

```text
for target in "armv7-linux-androideabi aarch64-linux-android"; do
  # Clean up the build tree
  git clean -x -f -d
```

```text
# Boot up the build system
./boot
```

```text
# Configure a GHC that targets $target
./configure --target=$target \
            --prefix=$PREFIX \
            --disable-large-address-space \
            --with-iconv-includes=$LIBICONV/$target/include \
            --with-iconv-libraries=$LIBICONV/$target/lib \
            --with-system-libffi \
            --with-ffi-includes=$LIBFFI/$target/include \
            --with-ffi-libraries=$LIBFFI/$target/lib
```

```text
# Create a mk/build.mk and set the BuildFlavour to quick-cross
sed -E "s/^#(BuildFlavour[ ]+= quick-cross)$/\1/" \
  mk/build.mk.sample > mk/build.mk
```

```text
  # Compile and install ghc
  make -j && make install
done
```

As this builds two cross compilers (for `armv7` and `aarch64`), this
will take approximately 60--120 minutes, depending on your hardware.
Once done, it should have installed `armv7-linux-androideabi-ghc` and
`aarch64-linux-android-ghc` into `/my/prefix/bin`.


## Compiling Hello World {#compiling-hello-world}

For Android we need to produce a hello world library, and call the
native code from an Android app.

The library `Lib.hs` contains a thin wrapper around `hello`, and exposes
a c function: `char* hello()`.

```text
module Lib where
```

```text
import Foreign.C (CString, newCString)
```

```text
-- | export haskell function @chello@ as @hello@.
foreign export ccall "hello" chello :: IO CString
```

```text
-- | Tiny wrapper to return a CString
chello = newCString hello
```

```text
-- | Pristine haskell function.
hello = "Hello from Haskell"
```

Assuming our Android application lives in `/path/to/HelloWorld`, we
create `/path/to/HelloWorld/app/hs-libs/armeabi-v7a` and
`/path/to/HelloWorld/app/hs-libs/arm64-v8a`.

We will make use of GHCs `-staticlib` flag has to produce a static
library that contains the `Lib.o` as well as all dependencies in a
single =.a= archive.

```text
aarch64-linux-android-ghc -odir arm64-v8a -hidir arm64-v8a \
   -staticlib -liconv -lcharset \
   -L/path/to/libffi/aarch64-linux-android/lib -lffi \
   -o /path/to/HelloWorld/app/hs-libs/arm64-v8a/libhs.a \
   Lib.hs
```

```text
armv7-linux-androideabi-ghc -odir armeabi-v7a -hidir armeabi-v7a \
   -staticlib -liconv -lcharset \
   -L/path/to/libffi/armv7-linux-androideabi/lib -lffi \
   -o /path/to/HelloWorld/app/hs-libs/armeabi-v7a/libhs.a \
   Lib.hs
```

/Note: The/=-liconv -lcharset= /and/=-L/path/to/libffi... -lffi=
_arguments are currently necessary, because ghc does not pass them
properly. The_ `libffi` _arguments are needed only if GHC is configured
with_ =--with-system-libffi=/./

We will start out with a fresh new android application with including
**_C++_** and _Kotlin_ support (you can also use Java, the example code
will be in Kotlin though), with an _Empty Activity_ named
`MainActivity`. For C++ use the Default Toolchain, and neither support
for _exceptions_ nor _rtti_ is needed.

In the `CMakeLists.txt` file, we need to tell CMake about our new
`libhs.a` and that we want to link against `libc`.

Adding the following two `find_library` statements:

```text
# find libc
find_library( c-lib
              c )
```

```text
# find libhs in /path/to/HelloWorld/app/hs-libs/<abi>,
# outside of cmakes root search path.
find_library( hs-lib
              hs
              PATHS ${PROJECT_SOURCE_DIR}/hs-libs/${ANDROID_ABI}
              NO_CMAKE_FIND_ROOT_PATH )
```

and including the found libraries in the final `target_link_library`
statement

```text
target_link_libraries( # Specifies the target library.
                       native-lib

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib}
                       ${c-lib}
                       ${hs-lib}
                       )
```

will instruct CMake to link the `native-lib` which contains our JNI
bridge to link against the `libhs` as well as `libc`.

Setting the `abiFilters` in the `app/build.gradle` file to

```text
android {
  ...
  defaultConfig
    ...
    ndk {
      abiFilters 'armeabi-v7a', 'arm64-v8a'
    }
  }
  ...
}
```

will tell Android Studio that we only have `armv7` and `aarch64` native
libraries. Adjusting the `native-lib.cpp` to read

```text
#include <jni.h>
#include <string>

#ifdef __cplusplus
extern "C" {
#endif
extern void hs_init(int * argc, char ** argv[]);
extern char* hello(void);
```

```text
JNIEXPORT void
JNICALL
Java_com_zw3rk_helloworld_MainActivityKt_initHS(
        JNIEnv *env,
        jclass /* klass */) {
    hs_init(NULL,NULL);
}
```

```text
JNIEXPORT jstring
JNICALL
Java_com_zw3rk_helloworld_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    return env->NewStringUTF(hello());
}
```

```text
#ifdef __cplusplus
}
#endif
```

will provide the `hello` prototype, which we can use in the
`stringFromJNI` method to call our `hello()` function. We also have the
`hs_init` prototype. This one will initialize the haskell runtime and
should be called **only once**, _prior to calling any haskell function_.

The `MainActivity` class from the `MainActivity.kt` then looks like
this:

```text
external fun initHS()
```

```text
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Example of a call to a native method
        initHS()
        val tv = findViewById(R.id.sample_text) as TextView
        tv.text = stringFromJNI()
    }

    /**
     * A native method that is implemented by the 'native-lib'
     * native library, which is packaged with this application.
     */

    external fun stringFromJNI(): String

    companion object {

        // Used to load the 'native-lib' library on
        // application startup.
        init {
            System.loadLibrary("native-lib")
            initHS()
        }
    }
}
```

_Note: as_
[_pointed
out_](https://www.reddit.com/r/haskell/comments/6e6fhq/a%5Fhaskell%5Fcross%5Fcompiler%5Ffor%5Fandroid/di98fwi/) _by_ [_/u/gergoerdi on_](https://reddit.com/u/gergoerdi)
[_/r/haskell_](https://www.reddit.com/r/haskell/comments/6e6fhq/a%5Fhaskell%5Fcross%5Fcompiler%5Ffor%5Fandroid/)
_/ `onCreate` /can be called_ \*/multiple times/\*/. The example code has
been updated to move/ `initHS` _from_ `onCreate` _to right after the_
=loadLibrary=/./

This is all that is all the source that is needed for our Hello World
Haskell Android app. The source can be found at
[zw3rk/hs-android-helloworld](https://github.com/zw3rk/hs-android-helloworld).

{{< figure src="https://cdn-images-1.medium.com/max/600/1*XSrHinpQhKfoMH4WLYWXLA.png" caption="Figure 1: Haskell running on an Android device" >}}


## Hello from Haskell {#hello-from-haskell}

Finally launching and running the application on the device, we are
greeted with **Hello from Haskell**.

While the utility of this application is certainly questionable it
illustrates the essential steps required to build, link and run an
android application calling a _native_ haskell function.

With this we should be well equipped to build the GHCSlave application
for android next and be able to also cross compile Template Haskell to
Android.