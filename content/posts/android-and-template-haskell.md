+++
title = "Android and Template Haskell"
date = 2017-05-31T00:00:00+08:00
draft = false
author = "Moritz"
+++

With the
[Haskell
Cross Compiler for Android](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-android-8e297cb74e8a) we built yesterday, and the knowledge how
to
[Cross
Compiling Template Haskell](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914), today we will build the slave process to
run on the Android device to allow to us the use of Template Haskell in
Android applications.


## Prerequisites {#prerequisites}

As we did for the [Raspberry Pi](http://amzn.to/2qb8k10), we need to
[build](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914)
`iserv-proxy`
[and
the](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914) `iserv`
[library](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914).
_Please refer to the instructions there if something is unclear._

After checking out the custom ghc branch:

```text
git clone --recursive git://git.haskell.org/ghc.git
cd ghc
git remote add zw3rk https://github.com/zw3rk/ghc.git
git fetch zw3rk
git checkout -b zw3rk/my-ghc
git reset --hard zw3rk/my-ghc
git submodule update --init --recursive
```

we need to build the `iserv-proxy` with the our regular compiler:

```text
ghc/iserv $ cabal install -flibrary -fproxy
```

and the `iserv` library with the cross compilers:

```text
ghc/iserv $ armv7-linux-androideabi-cabal install -flibrary
ghc/iserv $ arch64-linux-android-cabal install -flibrary
```


## Building GHCSlave for Android {#building-ghcslave-for-android}

Similar to the
[Android
application we built yesterday](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-android-8e297cb74e8a), we need to build a haskell library and
a wrapper to use the library from the Android app.

The code for the GHCSlave android app can be found in the
[zw3rk/ghc-slave](https://github.com/zw3rk/ghc-slave) repository in
the `android` folder.

Building the static library for android is a simple as

```text
aarch64-linux-android-ghc -odir arm64-v8a -hidir arm64-v8a \
   -staticlib -threaded -liconv -lcharset \
   -L/path/to/libffi/aarch64-linux-android/lib -lffi \
   -o app/hs-libs/arm64-v8a/libhs.a -package iserv-bin \
   app/src/main/hs/LineBuff.hs
```

```text
armv7-linux-androideabi-ghc -odir armeabi-v7a -hidir armeabi-v7a \
   -staticlib -threaded -liconv -lcharset \
   -L/path/to/libffi/armv7-linux-androideabi/lib -lffi \
   -o app/hs-libs/armeabi-v7a/libhs.a -package iserv-bin \
   app/src/main/hs/LineBuff.hs
```

Note that we need `-threaded` and the `iserv-bin` package. Threaded
because the `startSlave` function calls `forkIO` to start the slave in a
separate thread. And the `iserv-bin` package to embed it into the static
library. The `LineBuff.hs` is only necessary to set the haskell line
buffering and get instant feedback, when piping `stdout` and `stderr`
into a `TextView`.

With the libraries built for both architectures, starting the
application from android studio should present us with

{{< figure src="https://cdn-images-1.medium.com/max/800/1*47iW4gLArcnMW3NbKjghAw.png" caption="Figure 1: GHCSlave running on android device" >}}


## Compiling some Template Haskell {#compiling-some-template-haskell}

We will build a sample application that uses the `gitrev` package to
show the git revision it was built from in a `TextView`. Pretty similar
to the Hello World application from
[yesterday](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-android-8e297cb74e8a),
it will however use Template Haskell.

The application can be found at
[zw3rk/hs-hello-templatehaskell](https://github.com/zw3rk/hs-hello-templatehaskell).

First we will need to install the patched `gitrev` package

```text
git clone https://github.com/mobilehaskell/file-embed.git
(cd file-embed && armv7-linux-androideabi-cabal install)
(cd file-embed && aarch64-linux-android-cabal install)
```

Following the example from yesterday, and using the following `Lib.hs`

```text
module Lib where
```

```text
import Development.GitRev (gitHash)
import Foreign.C (CString, newCString)
```

```text
foreign export ccall "gitrev" cgitrev :: IO CString
```

```text
-- | turn $gitHash into a c string.
cgitrev = newCString $gitHash
```

and compiling it via the GHCSlave will yield the `libhs.a`.

```text
aarch64-linux-android-ghc -odir arm64-v8a -hidir arm64-v8a \
   -staticlib \
   -o app/hs-libs/arm64-v8a/libhs.a -package gitrev \
   app/src/main/hs/Lib.hs \
   -fexternal-interpreter \
   -pgmi $HOME/.cabal/bin/iserv-proxy -opti10.0.1.21 -opti500
```

```text
aarch64-linux-android-ghc -odir arm64-v8a -hidir arm64-v8a \
   -staticlib -liconv -lcharset \
   -L/path/to/libffi/aarch64-linux-android/lib -lffi \
   -o app/hs-libs/arm64-v8a/libhs.a -package gitrev \
   app/src/main/hs/Lib.hs
```

```text
armv7-linux-androideabi-ghc -odir armeabi-v7a -hidir armeabi-v7a \
   -staticlib \
   -o app/hs-libs/armeabi-v7a/libhs.a -package gitrev \
   app/src/main/hs/Lib.hs \
   -fexternal-interpreter \
   -pgmi $HOME/.cabal/bin/iserv-proxy -opti10.0.1.22 -opti500
```

```text
armv7-linux-androideabi-ghc -odir armeabi-v7a -hidir armeabi-v7a \
   -staticlib -liconv -lcharset \
   -L/path/to/libffi/aarch64-linux-android/lib -lffi \
   -o app/hs-libs/armeabi-v7a/libhs.a -package gitrev \
   app/src/main/hs/Lib.hs
```

_Note: due to a bug in how ghc currently expects the GHCSlave to find_
`libiconv` _and_ `libcharset=/, we invoke the compiler twice. The first
time compiling/ =Lib.hs` _into_ =Lib.o=/, and the second time providing
all the libraries that should be linked into the final archive. At that
point we don't need the/ `-fexternal-interpreter` _anymore, as we
already have an up-to-date copy of_ =Lib.o=/. This limitation should
soon be lifted. An alternative is to copy/ `libiconv=/,/ =libcharset=/,
and/ =libffi` _into the_ `app/hs-libs/<arch>` _folder for each folder
and link them similar to the_ `hs-lib` _in the_ =CMakeLists.txt=/./

With all that setup, we can simply launch the android application from
android studio and see the git hash as expected.

{{< figure src="https://cdn-images-1.medium.com/max/800/1*biUjD4l63sx5m8EBi9rHyg.png" caption="Figure 2: Hello Template Haskell app running on android device" >}}

With this we now have a fully functioning Haskell Compiler with Template
Haskell support. Another day we will have a look at doing something a
little more interactive than just displaying a string of text.