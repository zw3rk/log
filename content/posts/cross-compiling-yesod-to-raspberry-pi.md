+++
title = "Cross Compiling Yesod to Raspberry Pi"
date = 2017-05-27T00:00:00+08:00
draft = false
author = "Moritz"
+++

[As
we have seen](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914) we can now cross compile Template Haskell. The
[Yesod Web Framework](http://www.yesodweb.com/) is
[one](http://snapframework.com/) [o](https://www.spock.li/)f
[many](http://haskell-servant.readthedocs.io/en/stable/) haskell web
frameworks. By default it makes use of
[shakespeare](http://hackage.haskell.org/package/shakespeare)
templates, which happen to use Template Haskell.

This is supposed to be a case study of cross compiling haskell packages
to the [Raspberry Pi](http://amzn.to/2qb8k10). We will study the
issues that can arise with this rather recent development and how to
mitigate them. It will involve updating packages to make use of the new
Template Haskell facilities and lifting build constraints.


## The Sample Yesod Application {#the-sample-yesod-application}

The application we are trying to compile is a rather simple one from the
[Yesod book](http://www.yesodweb.com/book). You can follow along with
the source from the `cross-compiling-yesod` subfolder in the
[zw3rk/medium](https://github.com/zw3rk/medium) repository.

```text
git clone https://github.com/zw3rk/medium.git
cd medium/cross-compiling-yesod
```

The `src/Main.hs` looks as follows

```text
{-# LANGUAGE OverloadedStrings     #-}
{-# LANGUAGE QuasiQuotes           #-}
{-# LANGUAGE TemplateHaskell       #-}
{-# LANGUAGE TypeFamilies          #-}
import           Yesod
```

```text
data HelloWorld = HelloWorld
```

```text
mkYesod "HelloWorld" [parseRoutes|
/ HomeR GET
/page1 Page1R GET
/page2 Page2R GET
|]
```

```text
instance Yesod HelloWorld
```

```text
getHomeR :: Handler Html
getHomeR  = defaultLayout
            [whamlet|Hello World! <a href=@{Page1R}>Go to page 1!|]
getPage1R = defaultLayout
            [whamlet|Page 1 <a href=@{Page2R}>Go to page 2!|]
getPage2R = defaultLayout
            [whamlet|Page 2 <a href=@{HomeR}>Go home!|]
```

```text
main :: IO ()
main = warp 3000 HelloWorld
```

This small example uses Template Haskell with two custom
[QuasiQuotes](https://downloads.haskell.org/~ghc/latest/docs/html/users%5Fguide/glasgow%5Fexts.html#template-haskell-quasi-quotation)
`parseRoutes` and `whamlet`.


## Cross Compiling to Raspberry Pi {#cross-compiling-to-raspberry-pi}

We will use the
[The
Haskell Cross Compilation](https://medium.com/@zw3rk/the-haskell-cabal-and-cross-compilation-e9885fd5e2f) toolchain with Raspbery Pi Slave process as
set up in
[Cross
Compiling Template Haskell](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914) post.

Let's start simple and see how far we get by installing the
dependencies:

```text
arm-linux-gnueabihf-cabal install --dependencies-only --allow-newer
```

_We need the_ `--allow-newer` _as we are building with a development
compiler._


## Build-Type: Custom {#build-type-custom}

After a little while this will fail with

```text
Failed to install entropy-0.3.7
Build log ( ~/.cabal/logs/entropy-0.3.7.log ):
cabal: Entering directory 'entropy-0.3.7'
[1 of 1] Compiling Main             ( entropy-0.3.7/dist/arm-linux-gnueabihf/setup/setup.hs, entropy-0.3.7/dist/arm-linux-gnueabihf/setup/Main.o )
Linking entropy-0.3.7/dist/arm-linux-gnueabihf/setup/setup ...
entropy-0.3.7/dist/arm-linux-gnueabihf/setup/setup: entropy-0.3.7/dist/arm-linux-gnueabihf/setup/setup: cannot execute binary file
cabal: Leaving directory 'entropy-0.3.7'
```

for `entropy-0.3.7` and `stm-chans-3.0.0.4`. This is the result of
`built-type: custom` as discussed in the
[The
Haskell Cabal and Cross Compilation](https://medium.com/@zw3rk/the-haskell-cabal-and-cross-compilation-e9885fd5e2f) post.

A quick inspection of
[entropy/Setup.hs](https://github.com/TomMD/entropy/blob/master/Setup.hs)
reveals that the setup is required for haddock and detecting
`HAVE_RDRAND`. It is probably ok to try and compile this without the
custom setup.
[stm-chans/Setup.hs](http://community.haskell.org/~wren/stm-chans/Setup.hs)
does some haddock setup as well, which we do not need right now.

```text
cabal get entropy
cd entropy-*
sed -i.bak -E 's/([Bb]uild-[Tt]ype:[ ]+)Custom/\1Simple/g' *.cabal
arm-linux-gnueabihf-cabal install --allow-newer
```

```text
cabal get stm-chans
cd stm-chans-*
sed -i.bak -E 's/([Bb]uild-[Tt]ype:[ ]+)Custom/\1Simple/g' *.cabal
arm-linux-gnueabihf-cabal install --allow-newer
```


## hsc2hs: unsupported directive in cross compilation {#hsc2hs-unsupported-directive-in-cross-compilation}

We will also see the [zlib](http://hackage.haskell.org/package/zlib)
package to fail with:

```text
Configuring zlib-0.6.1.2...
Building zlib-0.6.1.2...
Preprocessing library zlib-0.6.1.2...
Codec/Compression/Zlib/Stream.hsc:1023
   directive const_str cannot be handled in cross-compilation mode
cabal: Leaving directory 'zlib-0.6.1.2'
```

The GHC user guide section for the `hsc2hs` utility lists `const_str` as
[not
supported in cross-compilation mode](http://downloads.haskell.org/~ghc/master/users-guide/utils.html#cross-compilation) as well.

[Reid Barton](https://github.com/rwbarton) kindly pointed out
[#9689](https://ghc.haskell.org/trac/ghc/ticket/9689), which provides
a proper patch to `zlib`, which is much better than this
[hack](https://phabricator.haskell.org/D3613) to enable `const_str` in
`hsc2hs`.

Sadly though it looks like the patch has never made it into `zlib`; I've
opened [pull/#14](https://github.com/haskell/zlib/pull/14) on behalf
of the original authors.

```text
git clone https://github.com/mobilehaskell/zlib
(cd zlib && arm-linux-gnueabihf-cabal install --allow-newer)
```

Should make the `zlib` package available.


## Requires External Interpreter {#requires-external-interpreter}

A few other packages (th-lift-instances, math-functions, will have
failed with

```text
Failed to install th-lift-instances-0.1.11
Build log ( ~/.cabal/logs/th-lift-instances-0.1.11.log ):
cabal: Entering directory 'th-lift-instances-0.1.11'
Configuring th-lift-instances-0.1.11...
Building th-lift-instances-0.1.11...
Preprocessing library th-lift-instances-0.1.11...
[1 of 1] Compiling Instances.TH.Lift ( src/Instances/TH/Lift.hs, dist/arm-linux-gnueabihf/build/Instances/TH/Lift.o )
ghc: this operation requires -fexternal-interpreter
cabal: Leaving directory 'th-lift-instances-0.1.11'
```

This indicates some Template Haskell compilation is needed. With
GHCSlave started on the Raspberry Pi at `10.0.1.1`, we retry this with

```text
arm-linux-gnueabihf-cabal install --allow-newer \
   --ghc-options="-fexternal-interpreter \
                  -pgmi $HOME/.cabal/bin/iserv-proxy \
                  -opti10.0.1.1 -opti5000"
```


## Requires File or Process IO {#requires-file-or-process-io}

Next the `shakespeare` package will fail with

```text
Preprocessing library shakespeare-2.0.13...
[ 1 of 18] Compiling Text.IndentToBrace ( Text/IndentToBrace.hs, dist/arm-linux-gnueabihf/build/Text/IndentToBrace.o )
[ 2 of 18] Compiling Text.MkSizeType  ( Text/MkSizeType.hs, dist/arm-linux-gnueabihf/build/Text/MkSizeType.o )
```

```text
Text/MkSizeType.hs:7:1: warning: [-Wunused-imports]
    The import of ‘Language.Haskell.TH’ is redundant
      except perhaps to import instances from ‘Language.Haskell.TH’
    To import instances alone, use: import Language.Haskell.TH()
  |
7 | import Language.Haskell.TH (conT)
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
[ 3 of 18] Compiling Text.Shakespeare.Base ( Text/Shakespeare/Base.hs, dist/arm-linux-gnueabihf/build/Text/Shakespeare/Base.o )
[ 4 of 18] Compiling Text.Shakespeare ( Text/Shakespeare.hs, dist/arm-linux-gnueabihf/build/Text/Shakespeare.o )
```

```text
Text/Shakespeare.hs:237:24: error:
    Ambiguous occurrence ‘readProcessWithExitCode’
    It could refer to either
       ‘Language.Haskell.TH.Syntax.readProcessWithExitCode’,
    imported from
       ‘Language.Haskell.TH.Syntax’ at Text/Shakespeare.hs:43:1-33
    or ‘System.Process.readProcessWithExitCode’,
    imported from ‘System.Process’ at Text/Shakespeare.hs:63:24-46
    |
237 |   (ex, output, err) <- readProcessWithExitCode cmd args input
    |                        ^^^^^^^^^^^^^^^^^^^^^^^
```

```text
Text/Shakespeare.hs:468:23: error:
    Ambiguous occurrence ‘getModificationTime’
    It could refer to either
       ‘Language.Haskell.TH.Syntax.getModificationTime’,
    imported from
       ‘Language.Haskell.TH.Syntax’ at Text/Shakespeare.hs:43:1-33
    or ‘System.Directory.getModificationTime’,
    imported from ‘System.Directory’ at Text/Shakespeare.hs:54:26-44
    |
468 |     mtime <- qRunIO $ getModificationTime fp
    |                       ^^^^^^^^^^^^^^^^^^^
```

This is due to the deliberate naming of process and file IO functions
identical to their `System.Process` and `System.Directory` counterparts.
This way packages that use process or file IO, which likely import
`Language.Haskell.TH.Syntax` as well, will fail and can be inspected for
adjustments.

After a
[minor
adjustment](https://github.com/mobilehaskell/shakespeare/commit/24c702f1e6caaf629785c926ed0376d33b003f71) to the shakespeare package, similar to the adjustments made
to
[file-embed](https://github.com/mobilehaskell/file-embed/commit/679b3690e1edf53986ea3af83e99115fdc32b75c)
and
[gitrev](https://github.com/mobilehaskell/gitrev/commit/958f5bf8fd5d16c22f282a96d54ad1f8ddae5240)
[the
other day](https://medium.com/@zw3rk/cross-compiling-template-haskell-7e38c00c2914). We can install shakespeare as well:

```text
git clone https://github.com/mobilehaskell/shakespeare.git
cd shakespeare
arm-linux-gnueabihf-cabal install --allow-newer \
   --ghc-options="-fexternal-interpreter \
                  -pgmi $HOME/.cabal/bin/iserv-proxy \
                  -opti10.0.1.1 -opti5000"
```


## The Final Stretch {#the-final-stretch}

Returning to compile the dependencies for our Yesod app:

```text
arm-linux-gnueabihf-cabal install --dependencies-only --allow-newer
```

The `http-api-data` package will fail with a `build-type: custom`
failure. And the `yesod` package will fail with _ambiguous occurrences_,
which require
[yet
another small adjustment](https://github.com/mobilehaskell/yesod/commit/76cc8918e9b3f5b064b2a679b5ebba17a1e21f80), and install just fine after that.

```text
git clone https://github.com/mobilehaskell/yesod.git
cd yesod/yesod
arm-linux-gnueabihf-cabal install --allow-newer \
   --ghc-options="-fexternal-interpreter \
                  -pgmi $HOME/.cabal/bin/iserv-proxy \
                  -opti10.0.1.1 -opti5000"
```


## Building and Running the Yesod Application on Raspberry Pi {#building-and-running-the-yesod-application-on-raspberry-pi}

Finally we are in the position to build out Yesod app now, while this
was clearly a bit involved, over time (and with patches being
upstreamed) this will become much easier.

Building the app now is as simple as

```text
arm-linux-gnueabihf-cabal configure
arm-linux-gnueabihf-cabal build \
   --ghc-options="-fexternal-interpreter \
                  -pgmi $HOME/.cabal/bin/iserv-proxy \
                  -opti10.0.1.1 -opti5000"
```

Transferring the `app` binary to the Raspberry Pi and running it:

```text
scp dist/arm-linux-gnueabihf/build/app/app pi@raspberrypi:
```

```text
ssh pi@raspberrypi 'LD_LIBRARY_PATH=. ./app'
```

Opening <http://raspberrypi:3000> shows the running Yesod instance

{{< figure src="https://cdn-images-1.medium.com/max/800/1*PiXJYEkfvP4nu5lxLRXyYg.png" caption="Figure 1: The Haskell Yesod Web Framework running on Raspberry Pi" >}}


## Conclusion {#conclusion}

We have seen that cross compiling Haskell to Raspberry Pi with Template
Haskell support is possible. It currently still does require a few
manual interventions. Those will become unnecessary over time, when
patches are integrated into the libraries. There has also been
[some progress](https://github.com/haskell/cabal/issues/2327) on
lessening the need for _build-type: custom_ when it is just needed for
[doctest](https://github.com/sol/doctest). If packages can be freed
from needing custom build types for haddock and doctest, they would be
easier to cross compile.

Finally you might have noticed that I moved all the _patched_ packages
to [github.com/mobilehaskell](https://github.com/mobilehaskell). I'd
like to invite you to join that organisation and publish _patched_ forks
there, so that we all can benefit, and be able to find the _patched_
packages in a central location.