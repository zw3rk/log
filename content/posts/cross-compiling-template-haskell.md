+++
title = "Cross Compiling Template Haskell"
date = 2017-05-25T00:00:00+08:00
draft = false
author = "Moritz"
+++

As laid out yesterday, there are some interesting questions pertaining
to
[Template
Haskell and Cross Compilation](https://medium.com/@zw3rk/template-haskell-and-cross-compilation-16b92f40c6ab). Today we will put all the pieces
together and cross compile template haskell to our
[Raspberry Pi](http://amzn.to/2qb8k10) _with file and process IO_!


## The External Interpreter {#the-external-interpreter}

[GHCs
External Interpreter](https://ghc.haskell.org/trac/ghc/wiki/Commentary/Compiler/ExternalInterpreter) splits GHC into two components the haskell
compiler `ghc` and the interpreter server `iserv`. Passing
`-fexternal-interpreter` to `ghc` will spawn an `iserv` instance and run
all interpreted code through it. `ghc` can instruct `iserv` to load and
link libraries as needed, and evaluate bytecode objects. `iserv` in turn
can query `ghc` for the current compilation environment during the
evaluation.


## Splitting iserv {#splitting-iserv}

For the cross compilation scenario, we need a to split `iserv` into two
parts, an `iserv-proxy` which serves as the `iserv` interface to GHC on
the _build_ machine. And an `GHCSlave` on the _host_ machine. This means
we need to move from

```text
.---------------.
| GHC <-> iserv |
'---------------'
  build machine
```

to a setup that looks more like

```text
    build machine            host machine
.-----------------------.   .------------.
| GHC <-> iserv-proxy <-+---+-> GHCSlave |
'-----------------------' ^ '------------'
       ^                  |
       |                  '-- communication via sockets
       '-- communication via pipes
```

_Note: I will not go into the technical details of the_ `iserv-proxy`
_and_ `iserv` _library to be used in the_ `GHCSlave` _in this post._


## Building GHC {#building-ghc}

In
[A
Haskell Cross Compiler for Raspberry Pi](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-raspberry-pi-ddd9d41ced94) we have seen how to build a
cross compiling GHC. As this is still work in progress and not all diffs
have been merged, or are updated and rebased onto GHCs master branch on
a regular basis, you will need a very recent copy (as recent as today!).

```text
git clone --recursive git://git.haskell.org/ghc.git
cd ghc
git remote add zw3rk https://github.com/zw3rk/ghc.git
git fetch zw3rk
git checkout -b zw3rk/my-ghc
git reset --hard zw3rk/my-ghc
git submodule update --init --recursive
```

Following the build instructions in
[A
Haskell Cross Compiler for Raspberry Pi](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-raspberry-pi-ddd9d41ced94) should yield an up to date
`arm-linux-gnueabihf-ghc`.

_Note: The hard reset is needed, as_ `zw3rk/my-ghc` _is periodically
force pushed. Please let me know of a better solution!_


## Building iserv {#building-iserv}

To build `iserv-proxy` we will use `ghc` (the GHC that builds on and for
the build machine), as the proxy needs to be able to run on the _build_
machine. The `iserv-bin` package, which contains the `iserv-proxy` is
part of the ghc tree, and can be found in the `iserv` subfolder. We can
simply build and install it by saying

```text
ghc/iserv $ cabal install -flibrary -fproxy
```

This should build and install `$HOME/.cabal/bin/iserv-proxy`.


## Building GHCSlave {#building-ghcslave}

Next we need the GHCSlave to run on the Raspberry Pi. For this we need
the `iserv` library, to link into GHCSlave. To do this we need the
`arm-linux-gnueabihf-cabal` we created in
[The
Haskell Cabal and Cross Compilation](https://medium.com/@zw3rk/the-haskell-cabal-and-cross-compilation-e9885fd5e2f).

```text
ghc/iserv $ arm-linux-genueabihf-cabal install -flibrary
```

Next, we will build `RPi/Slave.hs` from the `ghc-slave` repository:

```text
git clone https://github.com/zw3rk/ghc-slave.git
cd ghc-slave/RPi
arm-linux-gnueabihf-ghc Slave.hs
```

This should provide the `Slave` executable for the Raspberry Pi. Because
the `Slave` binary links against the `libffi` shared object we built
when building the `arm-linux-gnueabihf-ghc`, we need that on the
Raspberry Pi as well:

```text
scp path/to/libffi/arm-linux-gnueabihf/lib/libffi.so.7 \
    pi@raspberrypi:
scp Slave pi@raspberrypi:
```

On the Raspberry Pi, we will need a temporary folder `$HOME/slave`,
where GHCSlave store libraries and objects needed for the evaluation of
splices.

```text
pi@raspberrypi:~ $ mkdir $HOME/slave
```

With all this properly set up we can start the GHCSlave on the Raspberry
Pi

```text
pi@raspberrypi:~ $ LD_LIBRARY_PATH=$HOME ./Slave $HOME/slave 5000
```


## Compiling Template Haskell via GHCSlave {#compiling-template-haskell-via-ghcslave}

On to the exciting part! As we now have `iserv-proxy` on the build
machine, and `GHCSlave` on the host, we finally have a _fully
functioning cross compiler with Template Haskell support!_

Let us use [David Luposchainsky](https://github.com/quchen)
[poor
man's supercompiler](https://github.com/quchen/articles/blob/master/useful%5Ftechniques.md#poor-mans-supercompiler) as an example.

```text
-- File: TH.hs
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE BangPatterns #-}
module TH where
```

```text
import Language.Haskell.TH.Syntax
```

```text
-- Calculate the n-th Fibonacci number's last 10 digits in O(n).
fibo :: Integer -> Integer
fibo = (`rem` 10^10) . fibo' (0, 1)
      where fibo' (a,_) 0 = a
            fibo' (!a,!b) n = fibo' (b, a + b) (n - 1)

-- Wraps an Integer in(to) a Q Exp
wrapTH :: Integer -> Q Exp
wrapTH n = [| n |]

-- | Wrap a certain Fibonacci number in an expression
fiboTH :: Integer -> Q Exp
fiboTH n = wrapTH (fibo n)
```

and the `Main` module:

```text
-- File: Main.hs

{-# LANGUAGE TemplateHaskell #-}

import TH

myFibo :: Integer
myFibo = $( fiboTH (10^5) )

main = print myFibo
```

Compiling this should be as simple as

```text
arm-linux-gnueabihf-ghc -fexternal-interpreter \
    -pgmi $HOME/.cabal/bin/iserv-proxy \
    -opti10.0.1.1 -opti5000 \
    TH.hs Main.hs
```

assuming your Raspberry Pis IP is `10.0.1.1`.


## Compiling a Cabal Package that required Template Haskell {#compiling-a-cabal-package-that-required-template-haskell}

As
[we
have seen yesterday](https://medium.com/@zw3rk/template-haskell-and-cross-compilation-16b92f40c6ab), the
[file-embed](https://hackage.haskell.org/package/file-embed) package
as well as the [gitrev](https://hackage.haskell.org/package/gitrev)
package use Template Haskell (and the file system and `git` process).
The [initial patch](https://phabricator.haskell.org/D3502) to enable
file system access and process IO tried to do this transparently by
hooking into GHCs Runtime System. This proved to be rather inelegant. A
[revised patch](https://phabricator.haskell.org/D3608) extends the
Quasi class. The drawback is that packages need to be adapted to the new
Quasi class and be more explicit about the file system and process IO to
work with cross compilation.

With some
[minor](https://github.com/zw3rk/file-embed/commit/679b3690e1edf53986ea3af83e99115fdc32b75c)
[alterations](https://github.com/zw3rk/gitrev/commit/958f5bf8fd5d16c22f282a96d54ad1f8ddae5240)
to the file-embed and gitrev package, we can make them work with the new
features, let's install them.

```text
git clone https://github.com/zw3rk/file-embed.git
(cd file-embed && arm-linux-gnueabihf-cabal install)
```

```text
git clone https://github.com/zw3rk/gitrev.git
(cd gitrev && arm-linux-gnueabihf-cabal install)
```

So far so good. Now, let us try the
[contrived
example](https://github.com/zw3rk/medium/tree/master/template-haskell-and-cross-compilation) from
[yesterday](https://medium.com/@zw3rk/template-haskell-and-cross-compilation-16b92f40c6ab)

```text
git clone https://github.com/zw3rk/medium.git
cd medium/template-haskell-and-cross-compilation
```

```text
# configure the th-example package without shared
arm-linux-gnueabihf-cabal configure --disable-shared
```

```text
# build it with cabal
arm-linux-gnueabihf-cabal build \
   --ghc-options="-fexternal-interpreter \
                  -pgmi $HOME/.cabal/bin/iserv-proxy \
                  -opti10.0.1.1 -opti5000"
```

_Note: you will need to have the the_ `Slave` _process running on your
Raspberry Pi at IP_ =10.0.1.1=/./

This will produce
`dist/arm-linux-gnueabihf/build/th-example/th-example`. Copying it over
to the Raspberry Pi and executing it should _just work_:

```text
scp dist/arm-linux-gnueabihf/build/th-example/th-example \
    pi@raspberrypi:
```

```text
ssh pi@raspberrypi 'LD_LIBRARY_PATH=. ./th-example --version'
```

And yield `Version 32b8bbb75cf8910443397bfab6b194c8e6d36d47`.


## Current Limitations {#current-limitations}

As we saw, packages that use file or process IO, need to be adapted to
use the new more explicit API to work. It should also be noted that all
this is quite new code, and not all differentials have landed in master
yet. I am quite certain there will be bugs, however I'd like to
encourage you to try this out and provide feedback, PRs and bugreports!