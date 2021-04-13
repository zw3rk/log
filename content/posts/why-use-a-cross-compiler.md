+++
title = "Why use a Cross Compiler?"
date = 2017-05-18T00:00:00+08:00
draft = false
author = "Moritz"
+++

Since [obsidian.systems](https://obsidian.systems) and
[zw3rk](https://zw3rk.com) posted the
[Cross
Compilation survey](https://medium.com/@zw3rk/hello-world-a-cross-compilation-survey-890cb95029d7) (and
[results](https://medium.com/@zw3rk/cross-compilation-survey-results-3988ad1b677b)),
I have been writing about cross compiling to
[Raspberry Pi](http://amzn.to/2re1GLQ), and specifically
[building
a SDK for Raspbian Jessie](https://medium.com/@zw3rk/making-a-raspbian-cross-compilation-sdk-830fe56d75ba), and a
[Haskell
cross compiler](https://medium.com/@zw3rk/a-haskell-cross-compiler-for-raspberry-pi-ddd9d41ced94), one topic has been coming up repeatedly: _Why use a
Cross Compiler?_

For those who work in the mobile application space, cross compilation is
very likely an essential part of their day. iOS and Android apps are
naturally cross compiled, even though this fact is mostly hidden away.
In both cases the application is developed on a much more powerful
machine than the one it is finally deployed to (or tested on).


## Power Differences {#power-differences}

This leads us to the first reason for using a cross compiler. If the
machine that _builds_ the software is much more powerful than the one
that finally _hosts_ and runs the software, it make economical sense to
use the more powerful machine to do the compilation.


## Resource Constraints {#resource-constraints}

Another reason to use a cross compiler is the fact that the _target_
that is finally _hosting_ the software might be ressource constraint in
a way that prohibits or severely hinders the execution of the compiler.
While the ressources are perfectly fine for running the compiled
software, they might not be adequate to run the compiler.


## Environmental Restrictions {#environmental-restrictions}

A third reason for using a cross compiler could stem from the
restrictions put onto the environment that governs the _target_. If the
_target_ simply does not admit to running the compiler or necessary
toolchain to compile the software, the software must be compiled on a
different _build_ machine.

---

Alternatives to cross compilation include building in an emulated
environment that is sufficiently close to the _target_. The _build
environment_ is then identical or sufficiently similar to the _host
environment_. If the compiler in question does not support cross
compilation this might be the only option.

Finally with cross compilation the compiler now has to deal with two
different machines. The _build_ machine on which the cross compiler can
run, but the software built by the cross compiler can not run on. And
the _host_ machine on which the built software can run, but the cross
compiler can not. We will learn more about the challenges this presents
for a Haskell cross compiler, and especially for Template Haskell next
week.

Should you use a cross compiler or not? I don't believe there is a
general answer to this. It all comes down to the constraints as
mentioned above; the feasibility and complexity of setting up (or
obtaining) the SDK, the toolchain, and the compiler compared to setting
up a virtual (or emulated) sufficiently similar environment to run a
regular compiler in.