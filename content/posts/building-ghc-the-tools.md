+++
title = "Building GHC: The tools"
date = 2017-11-11T00:00:00+08:00
draft = false
author = "Moritz"
+++

After
[getting
to know package databases](https://medium.com/@zw3rk/building-ghc-the-package-database-50c37cf6ce33) we are going to look into the set of tools
that are used when building the Glasgow Haskell Compiler.


## alex & happy {#alex-and-happy}

[alex](https://www.haskell.org/alex/doc/html/index.html) is a tool to
generate lexical analysers in Haskell, and as such can turn =.x= files
into =.hs= files.
[happy](https://www.haskell.org/happy/doc/html/index.html) is a parser
generator, which can turn =.y= files into =.hs= files.


## ghc-pkg {#ghc-pkg}

We saw during
[getting
to know package databases](https://medium.com/@zw3rk/building-ghc-the-package-database-50c37cf6ce33). It is the command line interface to the
package databases.


## ghc-cabal {#ghc-cabal}

While cabal comes with the `Cabal` library and the `cabal` command line
tool in the `cabal-install` package. The build system needs an easy way
to just extract values from =.cabal= files and add some customizations
to configuring, copying, and registering packages via cabal. This is
what `ghc-cabal` is for. It produces the `package-data.mk` file, which
is a Makefile that contains information from the =.cabal= file, such as
it's Haskell Sources, C Sources, ... and as such provides the Make based
build system with sufficient information to build all the libraries.
_Yes, we do_ **_not_** _use_ `cabal` _to build GHC._


## deriveConstants {#deriveconstants}

This utility is used during the build process to generate header files
containing constant values (word sizes, ...) of the target platform and
when the compiler is built those are included to provide those target
specific information.


## hsc2hs {#hsc2hs}

When binding to C libraries, the process can be quite tedious and
`hsc2hs` tries to make this a bit easier. It is also used quite a lot
outside of GHC itself. It even has its own
[GHC
User's Guide entry](http://downloads.haskell.org/~ghc/master/users-guide/utils.html#writing-haskell-interfaces-to-c-code-hsc2hs).


## genapply & genprimopcode {#genapply-and-genprimopcode}

GHCs runtime system needs to apply functions of various arities when
evaluating a program. `genapply` produces the `AutoApply.cmm` file which
is part of the `rts` package. The `AutoApply.cmm` file defines the
standard closures used to represent application of functions of various
artities.

Functions that can not be implement in Haskell directly are called
Primitive Operations (PrimOps) while these functions are provided
natively by GHC, the `genprimopcode` utility uses a highlevel
description of the PrimOps in the `compiler/prelude/primops.txt.pp` and
produces the necessary plumbing around them: header files, curried
wrappers, haddock information.


## touchy {#touchy}

While unix systems have the `touch` command. Windows does not. As GHC
uses `touch` during building and compilation of Haskell files, `touchy`
is required on Windows as a substitute.


## unlit {#unlit}

As GHC has support for Literate Haskell, it needs to be able to extract
the Haskell source from a =.lhs= source file. `unlit` is the filter that
can stirp the literate part out of a =.lhs= file and just leave
the =.hs= part.

---


## compareSizes {#comparesizes}

When comparing two ghc builds, it can be of interest to see whether or
not the file size of haskell interface files (`.hi`) or objects (`.o`)
has increased or decreased. As such the compareSizes utility is not
essential to build GHC, but useful when trying to gauge what effect a
change has on interface and object file sizes.


## hp2ps {#hp2ps}

The `hp2ps` tool is also widely used. It turns a heap profile generated
by GHC into a (hopefully) easier to undertand graphical representation.


## ghctags {#ghctags}

`vim` and `emacs` can read so called `TAGS` files, which provide an
index to source-code definitions of function, variables, and other
points of interest.

`ghctags` is capable of generating such a file for Haskell sources. As
such it is also a not an essential utility to _build_ GHC.


## hpc {#hpc}

The `hpc` tool can be used to generate code coverage reports for Haskell
code. It even has it's own
[page on the haskell
wiki](https://wiki.haskell.org/Haskell%5Fprogram%5Fcoverage).

---

In conclusion I hope this was somewhat informative. And while not all
tools are strictly used to _build_ GHC. I believe this covers all the
utilities that are part of the GHC source tree.