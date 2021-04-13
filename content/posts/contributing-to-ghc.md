+++
title = "Contributing to GHC"
date = 2017-12-21T00:00:00+08:00
draft = false
author = "Moritz"
+++

## via Phabricator {#via-phabricator}

About two weeks ago I gave a talk on “Contriubting to GHC via
Phabricator” at the local haskell.sg Meetup. Sadly the microphone died
after a few minutes into the recording, and as such the video has no
audio for most of the talk and is pretty useless. As the topic might be
of interest to those who could not attend, I'll write down the content
of the talk as good as I can.


## Phabricator?! {#phabricator}

Let's start with what [Phabricator](https://phacility.com/phabricator)
is, Phabricator is the code review tool GHC uses. It was originally
developed at Facebook by Evan Priestley, who has since founded
Phacility, Inc where the development of Phabricator continues.

Phabricator is not just used at GHC, it is also used for other projects
like LLVM, the Wikimedia Foundation, FreeBSD, and a many more.

I will now try to demonstrate the usage of Phabricator for a simple
change (with many pictures!).


## Creating an account {#creating-an-account}

[phabricator.haskell.org](https://phabricator.haskell.org,), this is
the first screen you will be greeted with. We will proceed to open an
account so we can do something useful.
![](https://cdn-images-1.medium.com/max/800/1*98D8QgM96kR26RmecTu29w.png)

form. While you can register a new account, I would suggest to use on of
the available alternative account connectors.
![](https://cdn-images-1.medium.com/max/800/1*T%5FAi4I5Uu8gF4NfsVYyoEQ.png)

{{< figure src="https://cdn-images-1.medium.com/max/800/1*pB5drd%5FhMHv4ql%5FxtT0Tiw.png" caption="Figure 3: For this example, we'll use GitHub." >}}

will still be required to provide an email address.
![](https://cdn-images-1.medium.com/max/800/1*hFW1W4mA4Wc8GmK7yxFZCw.png)

code, or even submit our own code for review. So let's do this next.
![](https://cdn-images-1.medium.com/max/800/1*5pxUAznO8MkMchBMDNBGiQ.png)


## I have a change in my tree {#i-have-a-change-in-my-tree}

Phabricator is not really concerned how your local tree looks like. It
essentially only cares about differences. So you can go wild with your
local branch management; use git however you like.

For Phabricator try to think in patches (differences) instead of
commits.

For the following example, I've been working on cross compiler related
issues and while doing so, I usually accumulate lots of different fixes
in my branch.

to `ghc-prim`) somewhere in my history, which I'd like to get into
GHC proper.
![](https://cdn-images-1.medium.com/max/800/1*h3D0CslstS8LVgTMh3Ynxg.png)

and cherry-pick the relevant commit onto it. _You could do this however
you like. Just make sure you have a handle on the range for your patch._
![](https://cdn-images-1.medium.com/max/800/1*SMfG0YgMzD08oTWU96TFFQ.png)

`git diff origin/master` should tell us. // Note the stupid white space!
![](https://cdn-images-1.medium.com/max/800/1*YpGcEx4Ea-6zZ5bVW5kRRQ.png)


## Enter arcanist {#enter-arcanist}

Phabricator has a companion tool called =arc=anist:

> “arc --- arcanist, a code review and revision management utility”

To install it, we need to clone the following two repositories:

```text
$ git clone https://github.com/phacility/libphutil.git
$ git clone https://github.com/phacility/arcanist.git
```

Then add `arcanist/bin` to your `PATH`.


## Link arc with your phabricator account {#link-arc-with-your-phabricator-account}

Next we'll need to tell connect `arc` with our phabricator account on
[phabricator.haskell.org](https://phabricator.haskell.org). To do this
we need to run the follwing command from within the ghc repository

```text
ghc $ arc install-certificate
```

get it from.
![](https://cdn-images-1.medium.com/max/800/1*xQMFr5Lk1ledPuT5CuBFQg.png)

following page. Simply copying the API Token ...
![](https://cdn-images-1.medium.com/max/800/1*sTjPP5ViqgShSolz%5FACIfQ.png)

to properly link your haskell phabricator account with your ghc
repository. _This needs to be done only once!_
![](https://cdn-images-1.medium.com/max/800/1*q6vGYh9kZTTKJ6b02Dt3Ig.png)

the offset, but usually `arc` should figure out the `origin/master`
offset on its own.
![](https://cdn-images-1.medium.com/max/800/1*8331KRzuZkcEW-ngkJs7Jw.png)

wants to be helpful here and tell us that we have lots of untracked
files in our repository. If you are sure you have committed everything
you want to submit for review, it's safe to say =y= here.
![](https://cdn-images-1.medium.com/max/800/1*-a0Oq5aLt3dYEnt19tFNnQ.png)

asked to fill out some metadata about your patch.
![](https://cdn-images-1.medium.com/max/800/1*t0GBH2hEv6ULZISTxITApQ.png)

summary and added Ben Gamari as a reviewer. Let's hope he's not
too busy!
![](https://cdn-images-1.medium.com/max/800/1*p0CcJ2lUD7mMR0bVkB4-vQ.png)

the patch. And of course it found the stupid white space from before. It
is however helpful enough to ask us if we just want to have this change
(removal of white space) applied.
![](https://cdn-images-1.medium.com/max/800/1*C2CYo8o%5FkZQRisvHOHJIMQ.png)

`arc` apply the patch. Next it wants to know if we want to amend HEAD.
And yes it makes sense to just amend this into the last commit. So I'll
go with =y= here.
![](https://cdn-images-1.medium.com/max/800/1*JQTE3Y6mFFaZcWpCEOfnsg.png)

phabricator; and we got a Differential identifier back `D4253` in
this case.
![](https://cdn-images-1.medium.com/max/800/1*ydCaAV9fhtZQRz9unh9oew.png)

with this page.
![](https://cdn-images-1.medium.com/max/800/1*-vpABfoTXiE%5FwKvgmWuvxQ.png)

Reviewer(s) we specified.
![](https://cdn-images-1.medium.com/max/800/1*jz%5Fax2o79beab05lpZ5yYQ.png)

{{< figure src="https://cdn-images-1.medium.com/max/800/1*biQ927q0mwtJv53o4oKKKg.png" caption="Figure 21: And even further down we can see the patch we submitted." >}}

Let's do a self-review here.
![](https://cdn-images-1.medium.com/max/800/1*DvPak1JQsb95fE8fb%5FHLww.png)

displayed but not the gray dashed border. **It is not submitted.** _I
think phabricator UI could be a bit better._
![](https://cdn-images-1.medium.com/max/800/1*kovhfohaSfc3pI3ej3eDSw.png)

that our remarks are actually submitted and others can see them.
![](https://cdn-images-1.medium.com/max/800/1*WsJ8hTUgKXSU9FQNybtkAA.png)

the timeline.
![](https://cdn-images-1.medium.com/max/800/1*sJI9wCsB4ZhbicY-LbVCRg.png)

comment probably doesn't hurt and might help the next one reading
the “code”.
![](https://cdn-images-1.medium.com/max/800/1*zNrp6BeKAIBTL9FhKrYbRA.png)

new patch.
![](https://cdn-images-1.medium.com/max/800/1*fhhx-BPsB7QsNJoz2rwEVg.png)

previous patch is left, because we essentially rewrote everything. The
second commit reversed everything from the first after all.
![](https://cdn-images-1.medium.com/max/800/1*NWmF3B3NSw0DzDapaZvZmA.png)

{{< figure src="https://cdn-images-1.medium.com/max/800/1*bErZ%5FGLlYkLc3zhdAhjYQg.png" caption="Figure 29: Time for `arc diff origin/master` again." >}}

{{< figure src="https://cdn-images-1.medium.com/max/800/1*JYJ38fJbQXMZg4m8WjmD-Q.png" caption="Figure 30: And again we are greeted with the untracked files screen." >}}

because we are updating a differential and are not creating a new one.
Here we can specify what we changed. By default it will pull information
from the commit messages.
![](https://cdn-images-1.medium.com/max/800/1*qfVPtw%5FGX8K9s6v-2wljPw.png)

Differential.
![](https://cdn-images-1.medium.com/max/800/1*QuxjIyovQ6OR1YUl-NU8cQ.png)

{{< figure src="https://cdn-images-1.medium.com/max/800/1*Rsu8gjPkXZn8qOE5aDGeMg.png" caption="Figure 33: Looking at the timeline we see the new update item as well." >}}

displayed. We'll mark the comment as Done...
![](https://cdn-images-1.medium.com/max/800/1*WaKxob5gX0EQn9J%5F9VwjtQ.png)

actually submitted.
![](https://cdn-images-1.medium.com/max/800/1*evbg8FUMWZHt5%5FfI9rXWaw.png)

the timeline.
![](https://cdn-images-1.medium.com/max/800/1*HTgouq9HsH9t1ZAxnrv9lA.png)


## Where to go from here? {#where-to-go-from-here}

Now someone will have to come and review the code, and select the
“Accept” Action. If the code is accepted, you can run `arc land` \*if you
have commit access to GHC\*/. If you do not have commit access worry not,
someone with commit access will usually aggregate accepted diffs
run /=./validate= _on them and if that passes push them to the GHC
repository._

---

_If you like what you read and want to support further work like this,
feel free to support me via_
[_patreon_](https://www.patreon.com/zw3rk) _or send cryptocurrencies
to_

---

<a id="orgd01af18"></a>
[**Mobile Haskell (@mobilehaskell) |
Twitter**\\\\
/The latest Tweets from Mobile Haskell (@mobilehaskell). Advances in
using Haskell for mobile development Tweets
by.../twitter.com](https://twitter.com/@mobilehaskell)[[<https://twitter.com/@mobilehaskell>][]]

<a id="org4472ead"></a>
[**zw3rk (@zw3rktech) | Twitter**\\\\
/The latest Tweets from zw3rk (@zw3rktech). your friendly haskell
consultants/twitter.com](https://twitter.com/@zw3rktech)[[<https://twitter.com/@zw3rktech>][]]

<a id="orgb04e7ab"></a>
[**Moritz Angermann (@angerman\_io) |
Twitter**\\\\
/The latest Tweets from Moritz Angermann (@angerman\_io). Type masseur.
Singapore/twitter.com](https://twitter.com/@angerman%5Fio)[[<https://twitter.com/@angerman%5Fio>][]]