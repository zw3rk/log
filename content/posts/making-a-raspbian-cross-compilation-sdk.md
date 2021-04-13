+++
title = "Making a Raspbian Cross Compilation SDK"
date = 2017-05-11T00:00:00+08:00
draft = false
author = "Moritz"
+++

After setting up your
[Raspberry
Pi with Raspbian jessie lite](https://medium.com/@zw3rk/quick-headless-raspberry-pi-setup-52ad6dd312c4), this time we will now create the
necessary cross compilation Software Development Kit (SDK). The SDK will
contain the toolchain, headers and libraries to compile binaries
compatible with the Raspberry Pi on a different (more powerful) machine
(e.g. your laptop or desktop computer).

While the [Raspberry Pi 2](http://amzn.to/2qdnoiW) and
[Raspberry Pi 3](http://amzn.to/2qdnNCj) are `armv7` and `aarch64` the
old Raspberry Pi 1 and [Raspberry Zero](http://amzn.to/2pToU8W) are
`armv6`; to simplify matters
[Raspbian](https://www.raspberrypi.org/downloads/raspbian/) is
compiled for `armv6hf+vfpv2` to have a single distribution supporting
all. The target's triple is then `arm-none-linux-gnueabihf`.

We will therefore create a SDK targeting `armv6` and the programs built
with it should be compatible with all Raspberry Pis. The downside is
that programs built with this toolchain will not be able to take
advantage of advanced features only available in the Raspberry Pi 2 and

1.

It is instructive to build the SDK by hand to better understand the how
everything fits together, and be able to make educated adjustments to
the SDK's toolchain, headers and libraries. For our SDK we will focus on
`clang` from the [llvm project](https://llvm.org) as our underlying
compiler for our toolchain as we will later use GHC's LLVM cross
compilation backend, and want to keep things simple and consistent. You
can find a
[script to build
the SDK](https://github.com/zw3rk/scripts/blob/master/make-sdk) in our [zw3rk/scripts](https://github.com/zw3rk/scripts)
GitHub repository.


## The SDK {#the-sdk}

What is a SDK and why do we need one? When compiling on a more powerful
(or better supported) _build_ machine to a different one, we need a
**toolchain** that contains a **compiler** that can _target_ the machine
that will finally _host_ the resulting binary. The toolchain will also
contain a **linker**, to be able to link object files that are foreign to
the build machine as well as an **archiver**, and a tool to extract
symbols from those object files. In addition to the toolchain, we also
need access to the **headers** and **libraries** on the _host_, against we
want to link. Let us start out by creating a folder `raspbian-sdk` for
the SDK, with two subfolders `prebuilt` (which will hold the toolchain)
and `sysroot` (which will hold the headers and libraries).

```text
mkdir -p raspbian-sdk/{prebuilt,sysroot}
```


## The Compiler {#the-compiler}

As mentioned above, we'll use `clang` from the [llvm
project](https://llvm.org) as our compiler. You can obtain a copy from their
[release download website](http://releases.llvm.org/download.html). At
the time of writing _LLVM 4.0.0_ is the latest release. `clang` is a
multi-target compiler, which means that the same compiler can produce
object code for different architectures. This can be controlled via the
`--target` flag.

LLVM can be unpacked and installed into `raspbian-sdk/prebuilt` as
follows:

```text
curl -L -O http://releases.llvm.org/4.0.0/clang+llvm-4.0.0-x86_64-apple-darwin.tar.xz
```

```text
xz -d clang+llvm-4.0.0-x86_64-apple-darwin.tar.xz
tar xf clang+llvm-4.0.0-x86_64-apple-darwin.tar \
    -C "/path/to/raspbian-sdk/prebuilt" --strip-components=1
```

_Note: This will install the_ `clang+llvm` _for macOS, the commands for
linux would be similar._


## The Linker and Other Utilities {#the-linker-and-other-utilities}

With the compiler in hand, we can now produce object files =.o=, but
when combining multiple object files, these need to be linked together.
To illustrate this, assume two object files `a.o` and `b.o`. Object file
`b.o` refers to a function `f` in `a.o`. It does so via an external
symbol reference (by name). The linker will, when combining `a.o` and
`b.o`, look for the symbol with the specified name in `a.o` and resolve
the reference.

[GNU Binutils](https://www.gnu.org/software/binutils/) provide a suite
of tools containing not only one **linker** (_ld.bfd_), but a second more
modern one (_ld.gold_) as well. In addition to the linkers, binutils
also contain an **archiver** _(ar)_, a **symbol listing tool** (_nm_), and a
few other tools. The latest version can be obtained from
[their ftp server](http://ftp.gnu.org/gnu/binutils/), the latest
version as of this writing is _binutils-2.28_.

As this is a source distribution, you will need to build binutils
yourself.

```text
./configure --prefix="/path/to/raspbian-sdk/prebuilt" \
            --target=arm-linux-gnueabihf \
            --enable-gold=yes \
            --enable-ld=yes \
            --enable-targets=arm-linux-gnueabihf \
            --enable-multilib \
            --enable-interwork \
            --disable-werror \
            --quiet
make && make install
```

This should configure, compile and install the suite of binutils into
`/path/to/raspbian-sdk/prebuilt`. While binutils can be configured to be
multi-target, I advise against doing so, as it requires passing the
target to tools when there is ambiguity. This becomes more complicated
in cases where the tools are called indirectly.


## The Headers and Libraries {#the-headers-and-libraries}

We finally need to get hold of a copy of the headers and libraries from
the Raspberry Pi. We need those so that the compiler has access to the
headers and libraries available on the Raspberry Pi. /This also means,
that if you install new libraries (with their headers), you will need to
refresh your SDK; make sure to install the/ `-dev` _packages if needed._

To copy the headers and libraries from the Raspberry Pi to the build
machine, we will use `rsync`. It is not available by default in raspbian
jessie lite and needs to be installed:

```text
ssh pi@raspberrypi 'sudo aptitude install -y rsync'
```

With that in place, we can start to copy the data to our build system:

_Note: as_ [_Rob van den Bogaard_](https://medium.com/u/e51ad21a2143)
_kindly_
[_pointed
out_](https://medium.com/@robvandenbogaard/maybe-good-to-note-that-on-osx-the-built-in-rsync-command-doesnt-accept-multiple-remote-source-343773b59d8b)/, the/ `rsync` _that comes with macOS Sierra 10.12 does not
support multiple_ \*/remote sources/\*/. Using the/ `rsync` _installed
via_ `brew install rsync` _however does._

```text
mkdir sysroot
rsync -rzLR --safe-links \
      pi@raspberrypi:/usr/lib/arm-linux-gnueabihf \
      pi@raspberrypi:/usr/lib/gcc/arm-linux-gnueabihf \
      pi@raspberrypi:/usr/include \
      pi@raspberrypi:/lib/arm-linux-gnueabihf \
      sysroot/
```

Running `rsync` at a later time again, has the benefit of copying over
only the new data, to keep your local copy in sync with the one on the
Raspberry Pi. _You should run the_ `rsync` _command in the_ `sysroot`
_folder again, if you installed additional libraries on your Raspberry
Pi, against which you want to link._


## Using the SDK {#using-the-sdk}

Assuming we now have the following structure:

```text
├── prebuilt
│   ├── arm-linux-gnueabihf
│   ├── bin
│   ├── include
│   ├── lib
│   ├── libexec
│   └── share
└── sysroot
    ├── lib
    └── usr
```

_(After unpacking_ `clang+llvm` _into_ `prebuilt` _and setting_
`prebuilt` _as the target for_ =binutils=/)/

We can compile a simple `hello.c` with the following content

```text
#include <stdio.h>
```

```text
int
main(int argc, char ** argv) {
  printf("Hello World!\n");
  return 0;
}
```

via our cross compilation SDK for `arm-linux-gnueabihf` like so:

```text
export COMPILER_PATH=sysroot/usr/lib/gcc/arm-linux-gnueabihf/4.9
./prebuilt/bin/clang --target=arm-linux-gnueabihf \
                     --sysroot=./sysroot \
                     -isysroot ./sysroot \
                     -L${COMPILER_PATH} \
                     --gcc-toolchain=./prebuilt/bin \
                     -o hello hello.c
```

Finally copy `hello` over to the Raspberry Pi

```text
scp hello pi@raspberrypi:
```

and execute it

```text
ssh pi@raspberrypi ./hello
```


## Conclusion {#conclusion}

We have seen that building and using a cross compilation SDK is not as
simple as `clang --target arm-linux-gnueabihf -o hello hello.c`, but is
still something manageable. The tricky part is getting all the compiler
flags right.

Luckily we can hide all of them with a simple shell wrapper. Putting

```text
#!/bin/bash
BASE=$(dirname $0)
SYSROOT="${BASE}/../../sysroot"
TARGET=arm-linux-gnueabihf
COMPILER_PATH="${SYSROOT}/usr/lib/gcc/${TARGET}/4.9"
```

```text
exec env COMPILER_PATH="${COMPILER_PATH}" \
     "${BASE}/clang" --target=${TARGET} \
                     --sysroot="${SYSROOT}" \
                     -isysroot "${SYSROOT}" \
                     -L"${COMPILER_PATH}" \
                     --gcc-toolchain="${BASE}" \
                     "$@"
```

into `prebuilt/bin/arm-linux-gnueabihf-clang` and making it executable
with

```text
chmod +x prebuilt/bin/arm-linux-gnueabihf-clang
```

allows us to simply say

```text
prebuilt/bin/arm-linux-gnueabihf-clang -o hello hello.c
```

to compile `hello.c` on our _build_ machine, into an executable that can
run on the _host_.