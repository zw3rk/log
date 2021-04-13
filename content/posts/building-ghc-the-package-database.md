+++
title = "Building GHC: The package database"
date = 2017-10-30T00:00:00+08:00
draft = false
author = "Moritz"
+++

Over the next few posts, I will explore how the Glasgow Haskell
Compilers build system works. I will try not to go into too much detail
how make based build system, and focus on the high level process and how
to arrive at the final compiler.

As we will have to deal with package databases, let us take a look at
packages and package databases today.


## Packages {#packages}

While we usually build packages for use with GHC via `cabal`, the
packages GHC knows about are those registered in the known (to GHC)
package database(s). The global package database usually resides next to
the `ghc` binary and is called `package.conf.d`.

To GHC, a package consists of a package specification and a collection
of interface files =.hi= together with one or more archives =.a= of
objects, =.so= _ =.dylib= / =.dll= shared libraries and one merged =.o=
object file for use with GHCi in a folder named `$name-$version`. /(The_
`$name-$version` _is also known as the installed package id or
unit-id)_. The package specification will contain the package name and
version, exposed modules, libraries, dependencies, and other package
metadata. The details can be found in
[GHC
User's Guide](https://downloads.haskell.org/~ghc/latest/docs/html/users%5Fguide/packages.html).

Thus, if you don't have or don't want to use `cabal`, you could build
your own package from scratch. Just collect the interface files =.hi=
and the libary in a folder and write the package specification.


## Package Database {#package-database}

The package database, is the `package.conf.d` folder as mentioned above.
It contains all the package specification as `$name-$version.conf`.
These contain the information about the library, search paths, where to
find the archives and (shared) objects and more.

In addition to the =.conf= files, it also contains a cache file. If you
simply drop your package specification into the `package.conf.d` folder,
the package database won't see it. For this we need the `ghc-pkg` tool
to rebuild the cache.


## The =ghc-pkg= Tool {#the-ghc-pkg-tool}

To make dealing with `package.conf.d` easier, GHC comes with the
`ghc-pkg` tool, which provides a rich set of commands to interact with
the package database. For example, to update the cache after dropping a
specification into the database by hand, one would run

```text
$ ghc-pkg recache
```

to rebuild the `package.cache` file.

If you want to restrict the packages available to your GHC, you could
for example create your own empty database, and copy over packages from
a different database using:

```text
$ ghc-pkg --expand-pkgroot describe base | \
    ghc-pkg --global-package-db package.conf.d register -
```

This will read `base` from the global package db, dump the content of
it's specification to `stdout` and register that package in the database
located at `package.conf.d` by reading the specification from `stdin`
(via `-`).

The `--expand-pkgroot` is necessary because this will only copy the
specification, it will not copy the contents (interface files,
libraries, ...). If the package used package database relative paths, we
want to expand them, prior to registering the package in a different
package database.

You will need to use some `--force` if you intent to register a package
without it's dependencies in a new package database. Because GHC can
also deal with a stack of package databases, and you can control this
stack either via `GHC_PACKAGE_PATH` or the `-[no-]global-package-db`,
`-[no-]user-package-db` and `package-db` flags, your packages in one
database can have dependencies on packages in another database.


## Summary {#summary}

GHC knows about packages through package database(s). Packages do not
necessarily need to be created with `cabal`, we can create our own
package from scatch if we really wanted to. _(Though using_ `cabal` _or
the_ `Cabal` _library makes dealing with it much easier)._

`ghc-pkg` is the command line interface tool to interact with package
database.

Next time we will look at a few more of the tools used in building GHC,
to have the necessary foundation to build GHC.