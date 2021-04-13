+++
title = "What is New in Cross Compiling Haskell"
date = 2018-10-09T00:00:00+08:00
draft = false
author = "Moritz"
+++

## August & September Edition {#august-and-september-edition}

Due to a lot of traveling in September, I wan't able to write the August
update in September. So what happened in the last two month?

For the cross compilation pipeline through the `llvm-ng`, I've added
binary serializability of `cmm` to a custom `ghc` fork, that allows to
dump the `cmm` representation. This is helpful as it allows the
decoupeling from the code generator and the ghc front end. What usually
happens is that ghc reads a file, parses it turns it into an AST,
desuguares it, runs a set of optimizations and finally turns it into STG
before turning it into `cmm`. So far as `cmm` wasn't really binary
serializable, you had to plug your code generator in GHC, and have the
frontend run, and then call your code gen. With the opportunity to just
dump the `cmm`, this can be completely decoupled; it also means we can
profile the code generator (`cmm` to `object file`) in isolation.

On the minimal ghc distribution side, I've come to the conclusion that
`ghc` packaged with the following libraries: `rts`, `ghc`, `ghc-prim`,
`integer-gmp`, `integer-simple`, `base`, `array`, `deepest`, `pretty`,
`ghc-boot-th`, and `template-haskell` seems to be sufficient; this
notably dropped `Cabal`, which means that `Cabal` needs to be
bootstrapped. I'm not sure how I feel about that, or if I'd prefer to
ship a static `cabal` binary alongside.

I'd ideally prefer to get rid to `template-haskell` as well, however as
that is the same package `ghc` is linked against, not shipping it and
potentially reinstalling a different one would potentially break
anything that depens on TH; a way around this could be to use the
external interpreter only; in which case you could recompile the
external interpreter against your changed TH library.

On the topic of Template Haskell. The great HLint uses `{-# ANN ... #-}`
pragmas, which cause `ghc` to enable Template Haskell for that module.
After some discussion with Simon Marlow and Neil Mitchell, `ghc` will
likely learn to ignore the `{-# ANN ... "HLint: ... #-}` pragmas, and
provide an `{-# HLINT ... #-}` pragma.

Finally I didn't have much time to spend on `-target`. I'd like to get
the minimal `ghc` distribution working for that first as I believe it
would make `-target` easier; and require to ship much less libraries.