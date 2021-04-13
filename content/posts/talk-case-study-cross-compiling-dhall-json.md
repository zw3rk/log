+++
title = "Talk: Case Study: Cross Compiling dhall-json"
date = 2018-03-14T00:00:00+08:00
draft = false
author = "Moritz"
+++

## haskell.sg Meetup --- March 2018 {#haskell-dot-sg-meetup-march-2018}

At the [haskell.sg
March Meetup](https://www.meetup.com/HASKELL-SG/events/246341985/) I gave a short presentation on how to cross compile
[dhall-json](http://github.com/dhall-lang/dhall-json) to raspberry pi,
and the issues you might run into. I wanted to show that it is possible
to cross compile non-trivial Haskell packages today, and that issues
(ghc-head, `build-type`, and Template Haskell) have mostly trivial fixes
of which most can in principle be upsteamed.

The talk coincided with the release of GHC 8.4, **and** the release of a
new `zlib` package to hackage, which renders the need to fix `zlib`
unnecessary.

Note that while this works, the fix for `contravariant` is a rather
crude one. For a proper fix and the complete set of patches see
[contravariant#44](https://github.com/ekmett/contravariant/issues/44).

---

_If you like what you read and want to support further work like this,
feel free to support me via_
[_patreon_](https://www.patreon.com/zw3rk) _or send cryptocurrencies
to_

---

<a id="orgbf7781d"></a>
[**Mobile Haskell (@mobilehaskell) |
Twitter**\\\\
/The latest Tweets from Mobile Haskell (@mobilehaskell). Advances in
using Haskell for mobile development Tweets
by.../twitter.com](https://twitter.com/@mobilehaskell)[[<https://twitter.com/@mobilehaskell>][]]

<a id="org614aacf"></a>
[**zw3rk (@zw3rktech) | Twitter**\\\\
/The latest Tweets from zw3rk (@zw3rktech). your friendly haskell
consultants/twitter.com](https://twitter.com/@zw3rktech)[[<https://twitter.com/@zw3rktech>][]]

<a id="org1397938"></a>
[**Moritz Angermann (@angerman\_io) |
Twitter**\\\\
/The latest Tweets from Moritz Angermann (@angerman\_io). Type masseur.
Singapore/twitter.com](https://twitter.com/@angerman%5Fio)[[<https://twitter.com/@angerman%5Fio>][]]