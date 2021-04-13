+++
title = "GHC's Cross Compilation Pipeline"
date = 2017-05-19T00:00:00+08:00
draft = false
author = "Moritz"
+++

Today is going to be a slight bit more technical, and less direct
practical utility. We will look at the steps that GHC takes to cross
compiles code via its LLVM backend.

In
[GHC's
Compiler Commentary](https://ghc.haskell.org/trac/ghc/wiki/Commentary/Compiler/HscMain) we can see how the front end takes a Haskell file
and after _Parsing_, _Renaming_, _Desugaring_, it ends up in GHC's
[Core](https://ghc.haskell.org/trac/ghc/wiki/Commentary/Compiler/CoreSynType)
language. The Core is then processed repeatedly by the _Simplification_
pass before being translated into
[STG](https://ghc.haskell.org/trac/ghc/wiki/Commentary/Compiler/StgSynType)
and finally
[Cmm](https://ghc.haskell.org/trac/ghc/wiki/Commentary/Compiler/CmmType).
Cmm is the language from which the three code generation backends in GHC
take off.


## The Cross Compilation Backend {#the-cross-compilation-backend}

The LLVM code generator takes in Cmm, and turns it into
[LLVM intermediate representation](http://llvm.org/docs/LangRef.html).
The LLVM IR is then passed through the
[_LLVM optimizer_](http://llvm.org/docs/CommandGuide/opt.html), the
[_LLVM static compiler_](http://llvm.org/docs/CommandGuide/llc.html),
[_GHC's
LLVM Mangler_](https://github.com/ghc/ghc/blob/master/compiler/llvmGen/LlvmMangler.hs), before it is finally passed off to the _assembler_, and
ends up as object code.


## LLVM IR {#llvm-ir}

The LLVM intermediate representation can be written either in the
textual human readable version or as
[LLVM Bitcode](http://llvm.org/docs/BitCodeFormat.html). LLVM Bitcode
is a binary format, that is represented as a stream of bits. Values in
the Bitcode format do not necessarily need to align with byte
boundaries.

/GHC's LLVM code generator currently produces textual ir. As the textual
IR is not guaranteed to be stable across LLVM releases, this is one of
the reasons that GHC is usually tied to a specific LLVM release./


## LLVM optimizer {#llvm-optimizer}

The LLVM optimizer `opt` reads in LLVM IR writes LLVM IR after
performing a set of optimizations. The LLVM IR GHC uses GHC's custom
calling convention `ghccc`, which requires the `-mem2reg` pass to be run
by the optimizer, thus the backend always passes `-mem2reg` unless the
`-O<n>` flag that is passed from GHC to the optimizer is greater than
`0`. In which case the optimizer runs `-mem2reg` anyway.


## LLVM static compiler {#llvm-static-compiler}

The LLVM static compiler `llc` turns the LLVM IR produced by the LLVM
optimizer into assembly for the given target.


## GHC's LLVM Mangler {#ghc-s-llvm-mangler}

After the LLVM IR GHC produces is fed through LLVM's optimizer and
static compiler, the resulting assembly might need some special
attention. Therefore GHC passes the generated assembly through the LLVM
Mangler. The mangler currently ensures that `-dead_strip` has no effect
on Mach-O platforms (macOS, iOS, ...). Dead stripping on Mach-O
platforms breaks GHC's Tables Next To Code optimization; it requires
functions to carry prefix data. LLVM unconditionally
inserts =.subsections\_via\_symbols= into the assembly. This leads the
linker to believe that only code after _live_ function symbols needs to
be retained and it then strips away the prefix data, if the previous
symbol is considered _dead._ _This should not be needed with LLVM5
anymore! (_[_LLVM: D30770_](https://reviews.llvm.org/D30770)/)/

The mangler currently mangles two additional items: function to object
mangling for ELF, and AVX instruction rewrites to fix AVX stack spills.
For AVX GHC essentially lies to LLVM about the stack size being 32byte
aligned, but then needs to rewrite the aligned AVX instructions to their
unaligned counterparts.


## The Assembler {#the-assembler}

Finally the mangled assembly is turned into =.o= object code, which is
then handed of to the linker. On macOS `clang` is currently used as the
assembler instead of the system assembler.

---

That concludes our midlevel tour through the GHC's LLVM backend. _Please
note that I did not discuss the optional_ `Splitter=/, and optional/
=MergeForeign` _phases._