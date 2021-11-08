# Building Clang the Google way

## What?
Build an archive similar to the ones available from [CIPD](https://chrome-infra-packages.appspot.com/p/fuchsia/clang)

## Why?
Try and get Fuchsia built on a new platform.

## How?

### TL/DR

### Long version

#### Setting the stage
I've been playing around with Fuchsia since it went public a few years ago, periodically building the latest version and taking a look at how it evolved. Then my family grew bigger and I suddenly found myself with an unlikely source of spare time - 15 minutes now, 10 - in two hours, mixed in between changing nappies and taking care of the household. The highly irregular and ultimately quite short periods meant that no 'serious' work can be done. At the same time, I was preparing for a conference talk on Risc-V - slides available [here](https://github.com/zmeiresearch/HackConf2021_Why-Go-Risc-V-y). And one night probably around 3AM a tought occured - *Can I run Fuchsia, or at least Zircon, on Risc-V?*

Google quickly turned up an answer - back when it was still called Magenta, Zircon has been ported to Risc-V - a good start, but outdated by several years. At present, Zircon is part of the Fuchsia checkout, let's try building it using the native Fuchsia *way*. That's the kind of task that perfectly matches my sketchy free time - make a change, kick off a build, come back later, fix new failures and retry. And I was down the rabbit hole, only to find myself trying to build Clang the way Google does.

#### Context
When building Fuchsia some packages are downloaded as prebuilt archives. This is done for convenience, to cut down on build resource requirements (both time and space) as well as to reuse what's already done in another place. To do so, it relies on a tool called CIPD - short from Chrome Infrastructure Package Deployment. Although the tool is [open-source](https://chromium.googlesource.com/infra/luci/luci-go/+/refs/heads/main/cipd). CIPD officially supports only x64 and arm64 and obviously it cannot be of any help in my quest to Risc-V compilation. Which is fine, I'm used to cross-compilation ever since I was in the dev boards business - which is quite a long time ago.

What's annoying, however, is that Fuchsia build systems makes it seemingly easy to specify a new target architecture, but it quickly escalates once you understand that quite a lot of packages are actually prebuilt and often packaged in a particular way. Maybe I'll be doing a separate write-up with more details, but long story short - after a few days of building toolchains and messing around, I decided to tackle the problem at it's root and try doing what Fuchsia expects - build the packages using CIPD.

#### Details
While basic usage (e.g. downloading) of CIPD packages is unrestricted, there is a profound lack of information on how actually CIPD works and, more importantly, how packages are built. There *is* documentation, but it is usually centered at specific tools, is outdated or assumes one is a *googler* - which I am not.

What intially led to some confusion was what exactly CIPD does and what it does not. And building a package CIPD does **not** do. Rather, there is a separate set of tools, living under the [Chromium repository](https://chromium.googlesource.com/infra/luci/) that *seem* to be doing the building. Studiying the documentation, there should be a set of 'recipes' specifying how to build a package - which sounds reasonable. Now, where would one find the recipe for clang - and the other Fuchsia-required packages? Maybe they live in [lucy-recipes](https://chromium.googlesource.com/infra/luci/recipes-py/)? Nah, that would've been too obvious. Back to reading documentation, duckduckgoing/googling and when that did not work, grep-ing the fuchsia and chromium source code. And there was a lead:
```
ivan@balha:~/fuchsia$ rg recipes
...
tools/integration/fint/README.md
244:into the [recipes][fuchsia-recipes] repository so recipes can construct
275:[fuchsia-recipes]: https://fuchsia.googlesource.com/infra/recipes 
```

Aha! A quick `git clone https://fuchsia.googlesource.com/infra/recipes/ && find ./ -name "*clang*` turned out a `./recipes/contrib/clang_toolchain.py`. Studying the contents I was sure I'm on the right track - there are options to turn on/off LTO, use LLD on OSX and a few other Clang/LLVM-specific ones. Time to kick off this build!


