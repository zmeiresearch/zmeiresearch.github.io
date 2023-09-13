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

```
ivan@balha:~/Work/fuchsia-infra/recipes$ ./recipes.py run contrib/clang_toolchain enable_lto=thin
```

At first, I experienced a failure related to a missing git directory, fixed by running
```
ivan@balha:~/Work/fuchsia-infra/recipes$ mkdir ~/Work/fuchsia-infra/recipes/.recipe_deps/recipe_engine/workdir/cache/git
```

Next, the recipe assumes goma is available and complains with authentication/configuration failures. No command-line switch seems to exist, thus a quick manual patch got the job done:
```
ivan@balha:~/Work/fuchsia-infra/recipes$ git diff recipes/contrib/clang_toolchain.py
diff --git a/recipes/contrib/clang_toolchain.py b/recipes/contrib/clang_toolchain.py
index 7d68cf787..7df0d2fc4 100644
--- a/recipes/contrib/clang_toolchain.py
+++ b/recipes/contrib/clang_toolchain.py
@@ -268,16 +268,17 @@ def RunSteps(
     use_inliner_model,
     builders,
 ):
-    use_goma = (
-        not api.platform.arch == "arm" and api.platform.bits == 64
-    ) and not api.platform.is_win
+    #use_goma = (
+    #    not api.platform.arch == "arm" and api.platform.bits == 64
+    #) and not api.platform.is_win
+    use_goma = False
     if use_goma:
         api.goma.ensure()
         ninja_jobs = api.goma.jobs
         goma_context = api.goma.build_with_goma()
     else:
         ninja_jobs = api.platform.cpu_count
-        goma_context = contextlib.nullcontext()
+        goma_context = None

     # TODO: builders would ideally set this explicitly
```

Running the recipe command again resulted in 5 hours of compilation and testing (on a slow 4C/4T Ryzen 3 machine) which worked OK, only for the process to fail with a:
```
[E2021-11-09T01:33:50.197624+02:00 3124 0 annotate.go:273] original error: interactive login is required
#4 runtime/asm_amd64.s:1371 - runtime.goexit()
cas: failed to create cas client: failed to get PerRPCCredentials: interactive login is required
```

Never heard of CAS before. Googling does not produce meaningful results at first. Running the executable gives:
```
ivan@balha:~/Work/zmeiresearch.github.io/blog$ /mnt/phys2/home/ivan/Work/fuchsia-infra/recipes/.recipe_deps/recipe_engine/workdir/cache/cipd/infra/tools/luci/cas/git_revision%3Acefd07c708bfd0bb37362a0c90e53fa31b0f8793/.cipd/pkgs/0/_current/cas
Client tool to access CAS.

Usage:  cas [command] [arguments]

Commands:
  help      prints help about a command
  archive   archive dirs/files to CAS
  download  download directory tree from a CAS server.
  whoami    prints an email address associated with currently cached token
  login     performs interactive login flow
  logout    removes cached credentials
  version   prints the executable version


Use "cas help [command]" for more information about a command.

ivan@balha:~/Work/zmeiresearch.github.io/blog$ /mnt/phys2/home/ivan/Work/fuchsia-infra/recipes/.recipe_deps/recipe_engine/workdir/cache/cipd/infra/tools/luci/cas/git_revision%3Acefd07c708bfd0bb37362a0c90e53fa31b0f8793/.cipd/pkgs/0/_current/cas version
0.1

CIPD package name: infra/tools/luci/cas/linux-amd64
CIPD instance ID:  xiOY69QEW4Td0_DKahtTNA30_SiIwg20ei1En_APEUQC
```

Lovely, just another undocumented tool :) :
