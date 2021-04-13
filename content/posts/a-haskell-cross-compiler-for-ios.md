+++
title = "A Haskell Cross Compiler for iOS"
date = 2017-06-06T00:00:00+08:00
draft = false
author = "Moritz"
+++

So far we have built
[a
Haskell Cross Compiler for Raspberry Pi](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-raspberry-pi-ddd9d41ced94), as well as
[a
Haskell Cross Compiler for Android](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-android-8e297cb74e8a). To round this off, we will build a
cross compiler for iOS as well.

With the WWDC signaling the end of 32bit devices and the last 32bit
devices are the [iPad (4th
gen) and iPhone 5/iPhone 5C](http://jamesdempsey.net/ios-device-summary/), we will only build the 64bit cross
compiler. _Note that what Apple calls_ `arm64` _is called_ `aarch64`
_elsewhere. This is rather unfortunate._


## The SDK & LLVM {#the-sdk-and-llvm}

Apple ships the iOS SDK with Xcode. Hence a recent copy of Xcode from
the AppStore is required. As Apple does not ship the `opt` and `llc`
with Xcode, and GHC currently requires `opt` and `llc` from llvm4, we
need to obtain a copy from the LLVMs
[release download website](http://releases.llvm.org/download.html) as
well.


## Toolchain Wrapping {#toolchain-wrapping}

Apple provides the `xcrun` utility with automatically sets up the
toolchain for the tools we need. We however will still need to provide
the target prefixed aliases for better autotools interop. Credit for the
initial work on setting this up goes to the
[ghc-ios-scripts](https://github.com/ghc-ios/ghc-ios-scripts), we'll
use a slightly modified version:

```text
#!/bin/bash
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
 aarch64-apple-ios-clang|aarch64-apple-ios-ld)
  flags="--sdk iphoneos ${cmd} -arch arm64"
  cmd="xcrun"
  ;;
 aarch64-apple-ios-*|aarch64-apple-ios-*)
  flags="--sdk iphoneos ${cmd}"
  cmd="xcrun"
  ;;
 x86_64-apple-ios-clang|x86_64-apple-ios-ld)
  flags="--sdk iphonesimulator ${cmd} -arch x86_64"
  cmd="xcrun"
  ;;
 x86_64-apple-ios-*)
  flags="--sdk iphonesimulator ${cmd}"
  cmd="xcrun"
  ;;
 # default
 *-nm|*-ar|*-ranlib) ;;
 *) echo "Unknown command: ${0##*/}" >&2; exit 1;;
esac
```

```text
exec $cmd $flags "$@"
```

The universal `wrapper` can be obtained from the
[zw3rk/toolchain-wrapper](https://github.com/zw3rk/toolchain-wrapper)
repository. It contains not only the iOS part above, but also the
sections for Raspberry Pi and Android and can thus be used for all three
platforms.

Again we need to create the target prefixed aliases:

```text
for target in "aarch64-apple-ios x86_64-apple-ios"; do
  for command in "clang ld ld.gold nm ar ranlib cabal"; do
    ln -s wrapper $target-$command
  done
done
```

The
[bootstrap](https://github.com/zw3rk/toolchain-wrapper/blob/master/bootstrap)
script from the
[zw3rk/toolchain-wrapper](https://github.com/zw3rk/toolchain-wrapper)
repository will generate all aliases for Android, iOS and Raspberry Pi.


## Prerequisites {#prerequisites}

With the toolchain wrapper and aliases in `PATH` all we need to build
GHC is `ghc` and `cabal`. Using [homebrew](https://brew.sh) getting a
recent enough copy of `ghc` and `llvm-3.7` should be as easy as

```text
brew install ghc llvm@3.7
```

`llvm@3.7` will install `opt-3.7` and `llc-3.7`, which we need for the
bootstrap compiler. However homebrew installs `ghc` without the version
suffixes for `llc` and `opt`. This can simply be fixed by replacing

```text
("LLVM llc command", "llc"),
("LLVM opt command", "opt")
```

with

```text
("LLVM llc command", "llc-3.7"),
("LLVM opt command", "opt-3.7")
```

in the `settings` file, which in the case of `ghc` from homebrew is
located at

```text
/usr/local/opt/ghc/lib/ghc-8.0.2/settings
```

We also need `alex` and `happy`, which can be installed with `cabal`

```text
cabal install alex happy
```

As we did for Raspberry Pi and Android, we also need libffi for iOS

```text
git clone https://github.com/zw3rk/libffi.git
```

```text
cd libffi
```

```text
./autogen.sh
```

```text
CC="aarch64-apple-ios-clang" \
CXX="aarch64-apple-ios-clang" \
        ./configure \
        --prefix=/path/to/libffi/aarch64-apple-ios \
        --host=aarch64-apple-ios \
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
CC="x86_64-apple-ios-clang" \
CXX="x86_64-apple-ios-clang" \
        ./configure \
        --prefix=/path/to/libffi/x86_64-apple-ios \
        --host=x86_64-apple-ios \
        --enable-static=yes --enable-shared=yes
```

```text
make && make install
```

_Note: we need to use the_
[_zw3rk/libffi_](https://github.com/zw3rk/libffi) _fork for_ `-ios`
_support until the_
[_libffi/libffi#307_](https://github.com/libffi/libffi/pull/307) /pull
request has been merged into the official libffi repository. Or a new
autoconf release (latest 2.69 is from 2012) is cut and widely
available./


## Building GHC {#building-ghc}

With `opt` and `llc` from llvm4,=alex=, and `happy` in `PATH`

```text
export PATH=$HOME/.cabal/bin:$PATH
export PATH=/path/to/llvm4/bin:$PATH
export PATH=/path/to/wrapped-toolchain:$PATH
```

building GHC from the patched `my-ghc` branch

```text
git clone --recursive git://git.haskell.org/ghc.git
cd ghc
git remote add zw3rk https://github.com/zw3rk/ghc.git
git fetch zw3rk
git checkout zw3rk/my-ghc -b my-ghc
git submodule update --init --recursive
```

should require nothing more than

```text
# set paths
export PREFIX=/my/prefix
```

```text
export LIBFFI=/path/to/libffi
```

```text
for target in "aarch64-apple-ios x86_64-apple-ios"; do
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

and shiny new `aarch64-apple-ios-ghc` and `x86_64-apple-ios-ghc` should
appear in `$PREFIX/bin` after 60--120 minutes, depending on your
hardware.


## Compiling Hello World {#compiling-hello-world}

Similar to how we built the
[Hello
World library for Android](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-android-8e297cb74e8a), we are going to build the same Hello World
library and wrap it into an iOS application.

The `Lib.hs` lirbary with the following code

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

exports the `chello` C function, which we will be calling from the iOS
application to obtain a C string from Haskell. Nothing too exciting yet,
but it will demonstrate the basic interop.

Creating a simple _Single View Application_ for _iOS_ using _swift_ with
Xcode should provide the neccessary Application Template for our Hello
World application. _Using Objective-C would make calling_ `chello` /a
bit easier. However as Apple has been pushing swift for a while now,
we'll use swift./

We will need a bridging header to bring the C prototypes into swift. The
simplest way to do this is to add a new objective c file to the project,
e.g. `tmp.m`, which in turn will cause Xcode to ask if it should create
a bridging header. Answer yes and delete `tmp.h`. Add the prototypes we
need to the `helloworld-Bridging-Header.h`

```text
extern void hs_init(int * argc, char ** argv[]);
extern char * hello();
```

In the `AppDelegate.swift`, we will call `hs_init` when the application
did finish launching:

```text
func application(_ application: UIApplication,
  didFinishLaunchingWithOptions launchOptions:
  [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
    // Override point for customization after application launch.
    hs_init(nil,nil)
    return true
}
```

Adding a `Label` to the `Main.storyboard`, connecting the `IBOutlet` it
to the `ViewController.swift`, and setting it's `text` property to the
result of `hello`

```text
class ViewController: UIViewController {
  @IBOutlet weak var label: UILabel!
  override func viewDidLoad() {
    super.viewDidLoad()
    label.text = String(cString: hello())
  }
}
```

should be sufficient. Before we can build and run the application on an
actual device, we need to build the Haskell library and tell Xcode to
link against it.

Compiling into a static libraries for `aarch64` (device) & `x86_64`
(simulator)

```text
aarch64-apple-ios-ghc -odir arm64 -hidir arm64 \
   -lffi -L/path/to/libffi/aarch64-apple-ios/lib \
   -staticlib -o hs-libs/arm64/libhs.a hs/Lib.hs
```

```text
x86_64-apple-ios-ghc -odir x86_64 -hidir x86_64 \
   -lffi -L/path/to/libffi/x86_64-apple-ios/lib \
   -staticlib -o hs-libs/x86_64/libhs.a hs/Lib.hs
```

and turning them into a **universal library** with both architectures
combined

```text
lipo -create -output hs-libs/libhs.a \
  hs-libs/arm64/libhs.a hs-libs/x86_64/libhs.a
```

should provide the `libhs.a` static library in the `hs-libs` folder.
Linking it in Xcode requires to add it, together with _libiconv.tbd_, to
the **Link Binary With Libraries** section in the **Build Phases** tab of
the helloworld project in Xcode. As we can not build bitcode only
libraries with GHC, we also need to set **Enable Bitcode** in the **Build
Settings** tab to **No**.

Finally running the application on the device should present us with

{{< figure src="https://cdn-images-1.medium.com/max/800/1*3L%5F6ykur6vEOybbdjQBkgw.png" caption="Figure 1: Haskell running on iOS" >}}