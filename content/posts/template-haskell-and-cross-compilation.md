+++
title = "Template Haskell and Cross Compilation"
date = 2017-05-24T00:00:00+08:00
draft = false
author = "Moritz"
+++

[Over](https://medium.com/@zw3rk/quick-headless-raspberry-pi-setup-52ad6dd312c4)
[the](https://medium.com/@zw3rk/making-a-raspbian-cross-compilation-sdk-830fe56d75ba)
[last](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-raspberry-pi-ddd9d41ced94)
[two](https://medium.com/@zw3rk/the-haskell-cabal-and-cross-compilation-e9885fd5e2f)
[weeks](https://medium.com/@zw3rk/why-use-a-cross-compiler-92322ef46e32)
we have seen how to build a Haskell cross compiler for the
[Raspberry Pi](http://amzn.to/2qb8k10). We have also seen a high level
overview of
[Template
Haskell](https://medium.com/@zw3rk/template-haskell-75c7b67f9718). Today we will look at what Template Haskell means in the
context of cross compilation.

As [pointed
out yesterday](https://medium.com/@zw3rk/template-haskell-75c7b67f9718), Template Haskell requires that GHC is able to run the
object code for the Template Haskell function. This in turn requires GHC
to be able to load this code into memory and make it executable. Therein
leis the first problem; our compiler is a cross compiler and produces
code to run on the _host_, say Raspberry Pi, and not on the machine we
_build_ it on. Thus we can not natively load and run the Template
Haskell function on the same machine that GHC runs on, because our
_host_ is different from our _build_ machine.


## Existing Solutions {#existing-solutions}

In
[The
Haskell Cabal and Cross Compilation](https://medium.com/@zw3rk/the-haskell-cabal-and-cross-compilation-e9885fd5e2f) post, I've mentioned
[evil
splicer](https://git-annex.branchable.com/design/assistant/blog/day%5F236%5F%5Fevil%5Fsplicer/) and [ZeroTH](https://hackage.haskell.org/package/zeroth).
Both use a strategy where the Template Haskell splices are extracted
from the source, compiled on the _build_ machine, and the results are
then spliced back into their places. This however implies that the
splices are evaluated on the _build_ machine with libraries build for
the _build_ machine. Subtle differences between _build_ and _host_ can
potentially lead to incorrect results (e.g. 32 vs 64bit).

[GHCJS](https://github.com/ghcjs/ghcjs) is a Haskell to JavaScript
compiler that uses the GHC API. As such it is also a cross compiler as
its _target_ is a JavaScript runtime, while GHCJS itself does not run on
the JavaScript runtime. GHCJS however supports Template Haskell via its
[out
of process Template Haskell solution](https://github.com/ghcjs/ghcjs/wiki/Porting-GHCJS-Template-Haskell-to-GHC) for some time now. GHCJS does
this by running a [node.js](https://nodejs.org) server. GHCJS
transfers the compiled Template Haskell code to the server process,
links and evaluates it there and ship the result back to the GHCJS
process.

Since GHC 8.0.1, GHC has
[The
External Interpreter](https://ghc.haskell.org/trac/ghc/wiki/Commentary/Compiler/ExternalInterpreter), which provides support for a very similar
mechanism. If GHC is provided with the `-fexternal-interpreter` flag, it
will evaluate interpreted code in a separate process.

If we tried a similar approach to GHCJS with our Raspberry Pi (and we
will tomorrow), we would run into an issue that is transparent to GHCJS,
but not in our case. GHCJS runs the node.js server _on the same
machine_. This is important, because it means that the node.js process
sees _the same file system_ that GHCJS sees. It also can shell out to
any process GHCJS could shell out to, because the run on _the same
machine._

Therefore, if we transfer, link and run a Template Haskell function on
our _host_ (here: Raspberry Pi), it will not see the same file system or
be able to shell out to the same processes that GHC could. The Template
Haskell function will see the _hosts_ and not the _build_ machines
environment. This is contrary to what many packages that use Template
Haskell assume; they assume to operate in the same environment, can
access the same file system, and call the same commands the GHC process
can. For example the
[gitrev](https://hackage.haskell.org/package/gitrev) package, which
allows among others to embed the git repository hash into the program,
natively assumes that it can operate in the same environment in which
GHC is operating; the one that has `git` available and contains the
module that is being compiled, and into which the git hash is supposed
to be embedded.


## File and Process IO Example {#file-and-process-io-example}

An example where both file IO, and process IO might be used would be an
application that wants to reproduce the licenses of the libraries it
uses, and provide a version with a git hash to identify the exact commit
from which it was built. While we could use a separate file containing
the licenses, we opt to actually include the licenses _in the binary_.
We'll assume that our `Main.hs` is in a git repository and our licenses
are aggregated in a `LICENSES` file.

```text
{-# LANGUAGE TemplateHaskell, LambdaCase #-}
```

```text
module Main where
```

```text
import Development.GitRev (gitHash)
import Data.FileEmbed     (embedStringFile)
import System.Environment (getProgName, getArgs)
import System.Exit        (die)
```

```text
main :: IO ()
main = getArgs >>= \case
  ["--licenses"] -> putStrLn $(embedStringFile "LICENSES")
  ["--version"]  -> putStrLn ("Version " ++ $gitHash)
  _otherwise     -> do
    prog <- getProgName
    die $ "usage: " ++ prog ++ " [--licenses] [--version]"
```

While this is clearly some rather
[contrived
code](https://github.com/zw3rk/medium/tree/master/template-haskell-and-cross-compilation), this rather simple, innocent looking snipped can not trivially
be cross compiled. When GHC encounters the
`$(embedStringFile "LICENSES")` splice, it will build a bytecode object
(BCO), that calls `embedStringFile` with `"LICENSES"`. Thus loading the
`file-embed` library, and evaluating the BCO. The BCO requests a handle
on the `embedStringFile` function, and invokes it with `"LICENSES"`.
While this could execute perfectly fine on the _host_, the function
`runIO` is opaque to GHC, and GHC can't tell if the `IO` action will
_read/write files_ or _call a program_. GHC would simply try to read the
`LICENSES` file on the _host._ This was not our intention! The same
holds for the `$gitHash` splice, from which we expect to run `git` on
the _build_ machine and obtain the git hash of the _source directory_.


## Summary {#summary}

We have now seen that Template Haskell poses some interesting problems
for cross compilation that require GHC to be able to load, link, and
evaluate object code on the _host_. Due to the environmental differences
(file system, available programs, ...) the inability to execute Template
Haskell functions on the _build_ machine makes file and process IO
nonsensical.