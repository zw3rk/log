+++
title = "iOS and Template Haskell"
date = 2017-06-07T00:00:00+08:00
draft = false
author = "Moritz"
+++

Similar to
[Android
and Template Haskell](https://medium.com/@zw3rk/android-and-template-haskell-afb7cbf1bff7) we need wrap the GHCSlave (remote iserv) instance
into an application for iOS. To provide the Template Haskell evaluation
context on the iOS device.

With the [Haskell Cross
Compiler for iOS](https://medium.com/p/7cc009abe208/edit) from yesterday, we will now build the GHCSlave iOS
application.

/WARNING: Due to a bug in the x86\_64 linker code, Template Haskell does
not yet work with the iOS Simulator. I have marked the commands for the
simulator in italic, and will remove this warning, once the linker code
is fixed. Until then Template Haskell will only work for/on the device./


## Prerequisites {#prerequisites}

Again we need the to
[build](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914)
`iserv-proxy`
[and
the](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914) `iserv`
[library](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914).
_Please refer to the_ [_Raspberry Pi_](http://amzn.to/2qb8k10)
_instructions if something is unclear._

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
ghc/iserv $ aarch64-apple-ios-cabal install -flibrary
ghc/iserv $ x86_64-apple-ios-cabal install -flibrary
```

With this we now should have the `iserv-proxy` binary in `~/.cabal/bin`
and the `iserv-bin` library for both cross compilers.


## Building GHCSlave for iOS {#building-ghcslave-for-ios}

We need to build a static library and wrap it into a native iOS
application.

The code for the GHCSlave android app can be found in the
[zw3rk/ghc-slave](https://github.com/zw3rk/ghc-slave) repository in
the `iOS` folder.

In the same fashion as we built the
[universal
library](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-ios-7cc009abe208) with `lipo`, we will build a universal GHCSlave library as
well.

In the `GHCSlave/iOS/GHCSlave` folder we build the static library for
the iOS device and simulator with `-threaded` and link in the
`iserv-bin`.

```text
aarch64-apple-ios-ghc -odir arm64 -hidir arm64 \
  -staticlib -threaded \
  -lffi -L/path/to/libffi/aarch64-apple-ios/lib \
  -o hs-libs/arm64/libhs.a -package iserv-bin \
  hs/LineBuff.hs
```

```text
x86_64-apple-ios-ghc -odir x86_64 -hidir x86_64 \
  -staticlib -threaded \
  -lffi -L/path/to/libffi/x86_64-apple-ios/lib \
  -o hs-libs/x86_64/libhs.a -package iserv-bin \
  hs/LineBuff.hs
```

```text
lipo -create -output hs-libs/libhs.a \
  hs-libs/arm64/libhs.a hs-libs/x86_64/libhs.a
```

_Note: we need_ `-threaded` _as the_ `startSlave` _function call_
`forkIO` _to start the salve in a separate thread._

Building and running the application on a device should show:

{{< figure src="https://cdn-images-1.medium.com/max/800/1*-v9WylZoLP-X5fgB1AxKOw.png" caption="Figure 1: GHCSlave running on an iOS device" >}}


## Compiling some Template Haskell {#compiling-some-template-haskell}

As we did when compiling
[Template
Haskell for Android](https://medium.com/@zw3rk/android-and-template-haskell-afb7cbf1bff7), we will use the `gitrev` package to embed the git
revision in a label.

The sourcecode for this sample application can be found at
[zw3rk/hs-ios-hello-templatehaskell](http://github.com/zw3rk/hs-ios-hello-templatehaskell).

First we will need to install the patched `gitrev` package

```text
git clone https://github.com/mobilehaskell/file-embed.git
(cd file-embed && armv7-linux-androideabi-cabal install)
(cd file-embed && aarch64-linux-android-cabal install)
```

And similar to the
[zw3rk/hs-ios-helloworld](https://github.com/zw3rk/hs-ios-helloworld)
example from
[yesterday](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-ios-7cc009abe208),
we'll build a library with the following `Lib.hs`

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

compiling this into a **universal library** with

```text
aarch64-apple-ios-ghc -odir arm64 -hidir arm64 \
  -staticlib \
  -L/path/to/libffi/aarch64-apple-ios/lib -lffi \
  -o hs-libs/arm64/libhs.a -package gitrev \
  hs/Lib.hs \
  -fexternal-interpreter \
  -pgmi $HOME/.cabal/bin/iserv-proxy -opti10.0.1.23 -opti5000
```

```text
x86_64-apple-ios-ghc -odir x86_64 -hidir x86_64 \
  -staticlib \
  -L/path/to/libffi/x86_64-apple-ios/lib -lffi \
  -o hs-libs/x86_64/libhs.a -package gitrev \
  hs/Lib.hs \
  -fexternal-interpreter \
  -pgmi $HOME/.cabal/bin/iserv-proxy -opti10.0.1.24 -opti5000
```

```text
lipo -create -output hs-libs/libhs.a \
  hs-libs/arm64/libhs.a hs-libs/x86_64/libhs.a
```

and linking it in Xcode,
[as
we did yesterday](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-ios-7cc009abe208), will provide us with an application that displays
the git hash as expected.

{{< figure src="https://cdn-images-1.medium.com/max/800/1*XkzkNHjyQ9%5Fih7VY2iA3sQ.png" caption="Figure 2: Hello Template Haskell app running on iOS device" >}}

We now have a functioning Haskell Compiler with Template Haskell support
for iOS. _(Again for now without support for the simulator)_