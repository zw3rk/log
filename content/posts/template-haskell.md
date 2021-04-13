+++
title = "Template Haskell"
date = 2017-05-23T00:00:00+08:00
draft = false
author = "Moritz"
+++

After using Haskell for a while, running into
[_Template
Haskell_](http://hackage.haskell.org/package/template-haskell) become more and more likely. A
[reverse
dependency lookup for template-haskell](http://packdeps.haskellers.com/reverse/template-haskell) lists over one thousand
packages.

While I believe many have _used_ Template Haskell via
[aeson](http://hackage.haskell.org/package/aeson),
[language-c-inline](http://hackage.haskell.org/package/language-c-inline),
[lens](http://hackage.haskell.org/package/lens),
[wreq](http://hackage.haskell.org/package/wreq),
[yesod](http://hackage.haskell.org/package/yesod), or any of the other
packages, my gut feeling is that fewer have actually _written_ Template
Haskell code.

/This post is not intended to advocate the use of Template Haskell, but
rather to foster a better understanding for what it is, and how it can
be used./

Template Haskell [is
described](https://wiki.haskell.org/Template%5FHaskell) as a facility that adds **_compile-time_** _metaprogramming_
to Haskell. Metaprogramming is the ability of a language to treat code
as data, and write code to produce other code to be used in the program.
Other languages have (limited) metaprogramming facilities as well: e.g.,
C has macros via the C preprocessor, that allow to expand code at
compile time as well, C++ has preprocessor macros and templates, and the
languages of the Lisp family usually have macro systems and are known
for their _homoiconicity_ and metaprogramming capabilities.


## The C preprocessor {#the-c-preprocessor}

To use metaprogramming facilities we need a way to say _how_ we want to
generate code, and a way to specify _where_ to generate code. The C
preprocessor does this on a textual basis with a separate language. We
can `define` a macro:

```text
#define ADD_ONE(x) ((x)+1)
```

this will simply replace all occurrences of `ADD_ONE(x)` with `(x)+1`
prior to handing off the code to the compiler. For example:

```text
int x = ADD_ONE(1);
```

becomes

```text
int x = (1)+1;
```


## Template Haskell Quotes {#template-haskell-quotes}

Contrary to the C preprocessor, Template Haskell operates on the
abstract syntax tree (AST) level. Again we need a way to describe what
we want the compiler to generate, and a directive to describe where to
generate it. GHC expects a function that generates code to live in the Q
monad. The Q monad governs its capabilities. Thus our `add_one` function
needs to be of the form:

```text
add_one :: Int -> Q a
```

We are free to generate different kinds of Haskell syntax, e.g., Types,
Declarations, Patterns, Expression, ...; here we clearly want an
Expression:

```text
add_one :: Int -> Q Exp
```

The first interesting question is, how to build this `Exp` Expression
type. The
[template-haskell
Haddocks for expressions](http://hackage.haskell.org/package/template-haskell-2.11.1.0/docs/Language-Haskell-TH.html#t:Exp) are a good start. We end up with:

```text
add_one :: Int -> Q Exp
add_one x = pure $ InfixE (Just (LitE (IntegerL x)))
                          (VarE '(+))
                          (Just (LitE (IntegerL 1)))
```

We translate `x :: Int` into a _literal expression_ with an _integer
literal_ of `x`, `+` into a _variable expression_, and then combine
those using the `InfixE` constructor for _infix expression_.

There **has to be** an _easier_ way, otherwise no one would bother writing
Template Haskell in the first place. And of course there is. Template
Haskell comes with _quotation_ support. The idea is to _quote_ some
expression (or type, declaration, pattern, ...) and obtain the above
expression as simple as:

```text
add_one :: Int -> Q Exp
add_one x = [| x + 1 |]
```

There are also quoter for declarations, type, patterns, ... or you can
even
[define
your own quoters using](http://downloads.haskell.org/~ghc/latest/docs/html/users%5Fguide/glasgow%5Fexts.html#template-haskell-quasi-quotation) `-XQuasiQuotes`.

_See also:_
[_The
GHC User Guide on Template Haskell_](https://downloads.haskell.org/~ghc/latest/docs/html/users%5Fguide/glasgow%5Fexts.html#template-haskell)


## Template Haskell Splices {#template-haskell-splices}

So far we learned how to create a function that can generate a Q-action.
Ultimately we want to convert the generated expressions to actual code
in our program. GHC calls this splicing. We splice the result of calling
the function (e.g. `add_one`) into our program. A _splice_ is written as
`$x` or `$(...)`, where `x` is an identifier or =...= is an arbitrary
expression. Thus if our `add_one` function lives in our `Lib` module, we
can build the following `Consumer` module, which makes use of `add_one`,
as follows:

```text
{-# LANGUAGE TemplateHaskell #-}
module Consumer where

import Lib (add_one)

two :: Int
two = $(add_one 1)
```

GHC will _at compile time_ evaluate the _splice_ `$(add_one 1)`, and put
the result of evaluating `add_one 1` in the Q monad into its place.


## The Stage Restriction {#the-stage-restriction}

You might be curious as to why we (had to) placed `add_one` into its own
`Lib` module. This is often referred to as the _stage restriction_. The
functions we want to call in our splices need to be defined outside of
the module where they are used. This is necessary since they need to be
compiled first. GHC compiles the module which contains the functions we
want to use in our splice first to machine code, and when evaluating the
splice, builds a tiny bit of interpreted bytecode that wraps the
arguments and calls the function.


## The Q Monad {#the-q-monad}

I've mentioned the Q monad earlier and said it governs the
metaprogramming capabilities of the functions we call in our splices.
The `Quasi`
[type
class](http://hackage.haskell.org/package/template-haskell/docs/Language-Haskell-TH-Syntax.html#t:Quasi) describes the interface that we can make use of when
constructing Template Haskell functions.

Of special note should be the `qRunIO :: IO a -> m a`. Which is wrapped
by `runIO :: IO a -> Q a`. This function allows to execute _arbitrary_
IO actions // **at compile time!** This allows capabilities like reading
(e.g. [file-embed](https://hackage.haskell.org/package/file-embed))
and writing arbitrary files or running commands/processes (e.g.
[gitrev](https://hackage.haskell.org/package/gitrev)). While this
power can provide great utility, I believe we should be well aware of
this fact when compiling arbitrary third party code.