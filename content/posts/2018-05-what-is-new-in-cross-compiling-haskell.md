+++
title = "What is New in Cross Compiling Haskell"
date = 2018-06-06T00:00:00+08:00
draft = false
author = "Moritz"
+++

## May Edition {#may-edition}

With GHC 8.6.1 being released soon, the windows for patches was closing
fast. While not necessarily cross compilation related, I was still able
to get a [patch](https://phabricator.haskell.org/D4714) in the
infamous load command size limit issue on macOS.

Interestingly windows suffers from a similar issue. On windows we have a
command argument size limit of 32k character from Windows 7 onwards.
Cabals (and to that extend also Stacks, and all other tools that use
`Setup.hs`) tendency to put libraries into their own folders results in
one library search path per library. Thus once you have accumulated
enough dependencies in your transitive dependency closure, you will see
the `gcc.exe: error: CreateProcess: No such file or directory` (or
`realgcc.exe`) errors. Now you might be wondering why? Arn't we using
response files? And indeed we use response files to pass the arguments
to `gcc`. However `gcc` internally does not seem to use response files
for the library search paths when passing them off to `collect2` and as
such fail over. Of course we can
[hack this rather crudely](https://phabricator.haskell.org/D4762) in
`ghc`, but the real solution is to fix `gcc`.

Now on to cross compilation related news: I've spent some more time
looking at `-target` but got nothing useful to report yet. I also
started looking on the `llvm-ng` backend and making it ready for the
upcoming ghc 8.6.1 release. This included mostly adapting new code gen
paths for new primops in 8.6.1.

I'm also trying to speedup the `llvm-ng` backend to be less space and
time consuming and hope to be able to provide prebuilt cross compiler
binaries for 8.6.1 in a timely manner around the 8.6.1 alpha/beta
releases.

Finally a shout out to [obsidian.systems](https://obsidian.systems/)
for releasing [obelisk](https://github.com/obsidiansystems/obelisk)!