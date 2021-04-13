+++
title = "What is New in Cross Compiling Haskell"
date = 2018-01-09T00:00:00+08:00
draft = false
author = "Moritz"
+++

## December Edition {#december-edition}

In December I was finally able to provide new GHC cross compiler
[binary
distribution](https://medium.com/@zw3rk/relocatable-ghc-cross-compiler-binary-distributions-f55080b837b1) for iOS, Android and Raspberry Pi from macOS Sierra and
Linux. The only other post was
[Contributing
to GHC](https://medium.com/@zw3rk/contributing-to-ghc-290653b63147) via phabricator. That is not to say there has been no progress,
I just found very little time to write.

Axis Sivitz has been successful in cross compiling some haskell code
using the cross compiler, and
[highlighted
a few issues along the way](https://github.com/mobilehaskell/hackage-overlay/issues/5). He also ran into an issue with the
[missing
adjustor support](https://github.com/mobilehaskell/hackage-overlay/issues/6), which should be rectified with a set of new cross
compiler releases later this month.

I've also talked to [fastly](https://www.fastly.com/) regarding their
open source support offerings, and I'm happy to report, that downloads
from [hackage.mobilehaskell.org](http://hackage.mobilehaskell.org)
will soon be delivered via fastly's CDN. I hope this will provide
everyone with a better (faster) experience!

🎉 Happy New Year 🎉