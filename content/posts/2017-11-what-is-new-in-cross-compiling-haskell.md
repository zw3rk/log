+++
title = "What is New in Cross Compiling Haskell"
date = 2017-12-08T00:00:00+08:00
draft = false
author = "Moritz"
+++

## November Edition {#november-edition}

Wow, it's December already! So what happened last month? As I detailed
in the
[October
Edition](https://medium.com/@zw3rk/what-is-new-in-cross-compiling-haskell-b70decd4b3a6) I have been working on a
[hadrian fork](https://github.com/snowleopard/hadrian/pull/445), that
should allow us to build relocatable binary distributions for GHC, and
specifically for cross compilers. This work has seen quite some
improvement (in hadrian and cabal), and I am now able to build
relocatable binary distributions of GHC cross compilers and if there are
no more obstacles, the should be ready soon. So keep an eye out for
changes on
[hackage.mobilehaskell.org](http://hackage.mobilehaskell.org), or just
follow [@mobilehaskell](https://twitter.com/mobilehaskell) where I'll
announce updates when they are ready.

Also
[part
2](https://medium.com/@zw3rk/building-ghc-the-tools-3d170a4db06c) and
[part
3](https://medium.com/@zw3rk/building-ghc-the-stages-2c6cf6fc4b29) to complement the
[first
part](https://medium.com/@zw3rk/building-ghc-the-package-database-50c37cf6ce33) of _Building GHC_ were published. I hope they provide some
insight into how the GHC build system works. This naturally came up
while hacking on hadrian.

[Alp Mestanogullari](https://github.com/alpmestan) has started helping
out with the hadrian. To get hadrian into a state where it can produce
proper installable distributions. As this is easier with relocatable
binary distributions, he has taken my branch as a basis for this. I hope
we'll be able to merge chunks of the branch back into the upstream
hadrian soon.

I've also started accepting donations via Patreon or crypto currencies
(e.g. see the
[Building
GHC: The Stages](https://medium.com/@zw3rk/building-ghc-the-stages-2c6cf6fc4b29) post. As such if you want to support this endeavor,
you now can!

As we've now got recording gear for our
[haskell.sg](http://haskell.sg) Meetups, I hope to be able to present
more on cross compilation and have recording of it up afterwards. The
first two recordings of the November talk, head over to the
[haskell.sg
youtube channel](https://www.youtube.com/playlist?list=PLbjcAmsCYuS4xspQb5BHnIHhi4BrWteGz). I hope to get around uploading the December
recordings over the weekend. However due to the microphone dying during
the first talk, we might only have the second talk properly recorded.

On a final note: if you have something specific on you'd like me to
write about, let me know!