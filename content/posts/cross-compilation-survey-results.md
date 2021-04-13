+++
title = "Cross Compilation Survey Results"
date = 2017-05-09T00:00:00+08:00
draft = false
author = "Moritz"
+++

About two weeks ago together with
[obsidian.systems](https://obsidian.systems),
[we
asked](http://um.com/@zw3rk/hello-world-a-cross-compilation-survey-890cb95029d7) about cross compiling haskell and 229 people responded to our
survey.

The majority (85%) noted that they would like to use GHC as a cross
compiler but do not yet. The targets for cross compilation are **arm**
(66%), **aarch64** (64%) and **x86\_64** (42%) for **android** (62%), **linux**
(60%) and **iOS** (43%).

Build architecture is almost exclusively **x86\_64** (98%), with **linux**
(82%) being twice as popular as **macOS** (38%), and **windows** a third at
(16%).

Onto the reasons for not using GHC as a cross compiler. With **lack of
clear instructions** (80%), and **building the cross compiler is too hard**
(60%), a clear lack of documentation is showing. In addition to the lack
of **Template Haskell** (27%), there were also a number of **other** reasons
mentioned (13%), ranging from debugging, through specific bugs in
libraries, to better tooling support.

The **Other thoughts** question received 40 distinct answers. With offers
to help, mentioning of existing approaches
([ghc-ios](https://github.com/ghc-ios),
[ghc-cross-compiler-windows-x86](https://github.com/albertov/ghc-cross-compiler-windows-x86),
[HaLVM](https://github.com/GaloisInc/HaLVM)), additional features
(nix), and experience reports.

If you want to get involved (please do!), [send
us an email](mailto:hello@zw3rk.com)!

The results confirm that what we have been working on (Template Haskell
and Tooling) aligns with the needs of the community (or at least those
who participated in the survey). It also highlights the need for better
documentation, which we will try to address as well (e.g. see
[Building
iconv for android](https://medium.com/@zw3rk/building-iconv-for-android-e3581a52668f)).

Follow us on medium, [github](https://github.com/zw3rk),
[twitter](http://twitter.com/zw3rktech), and
[subscribe to the mailing list](http://eepurl.com/cGh3Mb).