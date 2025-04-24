---
layout: post
title: "Sometimes the yak needs a trim"
categories: platform
tags: "ocaml windows"
---
Since presenting Relocatable OCaml at [OCaml Dev Meeting]({% post_url 2025-03-28-ocaml-dev-meeting %}),
I have been playing [whac-a-mole](https://en.wikipedia.org/wiki/Whac-A-Mole)
with our CI systems, working towards getting finalised branches for the work
ready for upstreaming. Eventually, it got to me, and I realised it was possibly
time to come up with a better test environment for these changes.

[Yak Shaving](https://en.wiktionary.org/wiki/yak_shaving):

1. Any apparently useless activity which, by allowing one to overcome
   intermediate difficulties, allows one to solve a larger problem.
2. A less useful activity done consciously or subconsciously to procrastinate
   about a larger but more useful task.

I definitely fear falling into the second definition! But, the problem at hand:
refactoring tests which run on a diverse set of platforms to make them
property-based, rather than name-based (i.e. going from "this test fails on
macOS, FreeBSD and OpenBSD" to "this test fails if the assembler is LLVM’s
internal assembler \[which happens to be the case on macOS, FreeBSD and
OpenBSD\]”). The problem was it’s quite hard to get right, so I was ending up
fixing a series of apparent glitches, then “playing CI golf” (push it to the CI
service, see what you forgot). What I really needed was to be able to work on
the test harness, edit it on any of the systems and quickly see the effect on
any of the others.

I used [Syncthing](https://syncthing.net/) both to share the support shell
scripts and also to distribute the test harness (yes, yes - what was handy with
Syncthing was _automatic_ synchronisation, which is why I didn't use Unison).
Syncthing insists on blatting files into the directories it’s synchronising,
which means I couldn't quite do what I wanted and sychronise the Git checkouts
directly (it would be _so_ lovely if Git had the ability to mark some files as
both ignored and _never cleaned_…). However, a certain amount of glue was
needed to kick off the builds, so it wasn’t too bad to have to hardlink the test
harness somewhere else for Syncthing to do it’s magic with. After pleasingly
little `sh` hacking:

![Roasting a laptop with 8 builds of OCaml!]({{ site.url }}/assets/2025-04-15/2025-04-15-screens.png)

The top is 5 different Windows configurations running in tmux (which needs
mintty sadly, as Cygwin’s tmux and Windows Terminal really don't seem to agree):
that's testing MSVC with `clang-cl`, vanilla MSVC, mingw-w64 in x86\_64 and i686
and then finally Cygwin itself. The bottom right is two builds of Linux running
in WSL on the same machine (testing a normal build and a static build). On the
bottom left is an SSH session to a Hyper-V VM running FreeBSD (also on the same
machine!) and then an SSH tunnel to the Mac Mini that lives on the desk in my
[Tarides](https://tarides.com) office. I then have a script which can be fed a
commit sha and additional configuration options, and all 9 of them then pick up
the instruction, rebuild the compiler and run the test harness. When something
fails, either the test harness can be edited separately, or the affected machine
can be broken out of the script and debugged - but as the test harness gets
updated, Syncthing redistributes it to the other machines and they immediately
re-run it.

Unsurprisingly, it was much more efficient to use than the CI golf - especially
when then testing individual commits with different build configurations. The
noise of the CPU fans is another matter, but I’m fortunate enough to have a new
workstation arriving fairly soon, so at least next time my poor laptop won’t
have to do all the work.

Conclusion of the week: occasionally the yak may need at least a trim!
