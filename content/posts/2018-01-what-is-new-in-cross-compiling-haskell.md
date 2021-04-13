+++
title = "What is New in Cross Compiling Haskell"
date = 2018-02-05T00:00:00+08:00
draft = false
author = "Moritz"
+++

## January Edition {#january-edition}

After setting up
[hackage.mobilehaskell.org](http://hackage.mobilehaskell.org) with S3
and fastly backing
[in
December](https://medium.com/@zw3rk/what-is-new-in-cross-compiling-haskell-759adaa7e1c), I automated the overlay generation in January. This means
that any patches pushed to
[the overlay
repository](https://github.com/mobilehaskell/hackage-overlay/) will be automatically reflected in the overlay after a few
minutes. Now all that's missing is patches for common packages!

I've also managed to [write
a patch](https://github.com/haskell/cabal/pull/5018) for `cabal` so that `new-build` properly propagates
`--with-PROG` flags into dependencies. This should make it work much
better for cross compilation, once the patch is merged. I'm hopeful this
will happen soon.

Finally I've argued for the use (and improved) overlays in the
[SLURP](https://github.com/haskell/ecosystem-proposals/pull/4) and
[Uncurated
Hackage Layer](https://github.com/haskell/ecosystem-proposals/pull/6) proposals, because I believe overlays are an underused
feature that could provide us with solutions to the perceived problems
(as I understand them) and offer a unified interface.