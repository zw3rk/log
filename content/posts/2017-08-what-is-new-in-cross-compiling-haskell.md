+++
title = "What is New in Cross Compiling Haskell"
date = 2017-09-05T00:00:00+08:00
draft = false
author = "Moritz"
+++

## August Edition {#august-edition}

The little time I manged to carve out for this project in August has
been spent on building a new llvm backend. Or more precisely to start
integrating the llvm bitcode backend I worked on last year into ghc.

The current status of the `-fllvmng` backend is that it can compile GHC,
but fails to validate. I believe most of these validation failures
should be fairly trivial so solve. Unless we run into
[interesting bugs](http://phabricator.haskell.org/D3904) again.

I dont believe I will mange to get much done in terms of actual code
this week, as I'm attending ICFP in Oxford. If you are around, please
say hi!

I hope you are excited for the posts to come! As always you can follow
us here on medium, on [github](https://github.com/zw3rk), or on
[twitter](http://twitter.com/zw3rktech).