+++
title = "What is New in Cross Compiling Haskell"
date = 2018-05-03T00:00:00+08:00
draft = false
author = "Moritz"
+++

## March & April Edition {#march-and-april-edition}

Due to some travel in April, I wasn't able to write up an update for
what happened in March. So today we'll have a combined update for March
and April.

Alp Mestanogullari has finished
[hadrian#531](https://github.com/snowleopard/hadrian/pull/531), which
was a clean up of my earlier
[hadrian#445](https://github.com/snowleopard/hadrian/pull/445). As
such we now build relocatable GHCs with hadrian by default! 🎉

I have been making progress on the Windows front to the point where we
don't even need a windows installation anymore, and run the iserv slave
via WINE. More on that soon!

As I've had to work a lot more with nix recently, I've come to hit the
limitations of the current Haskell infrastructure in nix with respect to
cross compilation. Specifically the flattening of conditionals
(os/arch/flags) that `cabal2nix` does. This resulted in
[some](https://github.com/angerman/Cabal2Nix)
[new](https://github.com/angerman/haskell.nix)
[tooling](https://github.com/angerman/hackage.nix) and a bit
[more](https://github.com/angerman/stackage.nix). This is in its early
stages, and I will expand on this properly once it has matured a little.

Finally I started looking into adding `-target` to GHC. This will
certainly be a very long road, but I think we do have most of the
necessary bits in place to make GHC multi-target. It will however
require not only changes to GHC, but also to the build system, to build
the relevant libraries in the right places, as well as to the tooling
around.