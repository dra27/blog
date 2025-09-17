---
layout: post
title: "Relocatable OCaml - from concept to demo to PRs"
categories: platform
tags: "ocaml windows"
---
If you give vapourware long enough, it condenses into an actual ware! An idea hatched between rehearsing Handel arias in the outskirts of Munich just over six years ago to make running OCaml's programs a little less surprising is now [132 commits of reality](https://github.com/ocaml/ocaml/pull/14247).

Relocatable OCaml started out as a solution to a few thorny problems with the darker corners of OCaml. The somewhat grander fast opam switches [referred to in the eventual RFC](https://github.com/ocaml/RFCs/pull/53) followed later.

In a nutshell, Relocatable OCaml unlocks the ability to _reliably_ distribute pre-compiled binaries for a given system. It means, for example, that if you and I are both running the same version of Ubuntu, and we both clone and build OCaml in `~/ocaml` and configured to install in `~/ocaml/install`, we'll both get **exactly** the _same compiler_, even if our user names - and consequently the directory we're installing to, differ.

There's a demo of how to try out Relocatable OCaml [on Discuss](https://discuss.ocaml.org/t/relocatable-ocaml/17253), but I thought it might be interesting to muse for a paragraph or two on why I think it ought to be done the way I've done it, and to a great extent why that's made it take quite so long to get it to a finalised state.

In November 2019, I posed three "problems" with OCaml installations, all based on real-world experience:
1. Bytecode programs were susceptible to loading the wrong C support libraries. The example I gave was a system where running `camlp4 -v` (to report the version of `camlp4` in an opam switch) instead complained about `dllunix.so` and a missing `caml_sigmask_hook` function.
2. Executables produced by the bytecode compiler stop working if directories get renamed.
3. Both compilers (`ocamlopt` and `ocamlc`) can fail to find the OCaml Standard Library.
and to this we can add a fourth:
4. It takes several minutes to start an OCaml project while waiting for `opam` to build OCaml from sources.

**All of these problems can be solved by changing the workflow!**

The first problem is comes about from `opam` abusing `CAML_LD_LIBRARY_PATH` (tracked in [ocaml/opam-repository#16406](https://github.com/ocaml/opam-repository/issues/16406)) and can be fixed by, er, not abusing `CAML_LD_LIBRARY_PATH`.
The second problem can be fixed by building bytecode executables with `-custom` or `-output-complete-obj` so that they include their interpreter (in fact, that's also a solution to the first problem).
The third problem comes about from the `OCAMLLIB` environment variable and can likewise be solved by simply not leaving bad or empty values of `OCAMLLIB` kicking around.
The fourth problem comes about because OCaml development - in common with many other languages - encourages having a development environment per-project, and since `opam` 2.0's "local switches" were introduced, the common workflow is to have an entire OCaml compiler installation with each project. This problem could be fixed by sharing the compiler between installations and configuring it appropriately for each project.

So why don't we just do that?

Papercuts. The problem is that each and every developer has to know all these things and make the connection between an obscure in front of them (for example `Error: Unbound module Stdlib` or `undefined symbol: caml_sigmask_hook`) and a seemingly unconnected change to their build or workflow.

**[Let's simplify!](https://en.wikipedia.org/wiki/Fluxx)**

As the test harness I added in [ocaml/ocaml#14014](https://github.com/ocaml/ocaml/pull/14014) shows, OCaml has a lot of different linking and loading mechanisms. It's a very reasonable approach instead of fixing them all to try to reduce the number we have to worry about.

The most complex part of Relocatable OCaml is the second PR ([ocaml/ocaml#14244](https://github.com/ocaml/ocaml/pull/14244)), which is concerned with determining where the OCaml Standard Library is located relative to an application. In particular, applying that knowledge at link time to all of these various linking mechanisms is likewise complex. The approach I've advocated preserves the status quo for the many tools and builds which implicitly need to know where the OCaml Standard Library is (in case you were wondering, pretty much all PPXs rely on it). Another approach might instead be to come up with a different compiler library design so that the location of the Standard Library becomes somewhat less of a concern _to the linker_!

However, to those papercuts, we should also acknowledge the elephant in the blog post, which is well-documented in [XKCD 927](https://xkcd.com/927/) - _new_ workflows brought in to fix old problems will just create _new_ problems.

I strongly contend that we must either _fix the old problems_ or, if we're going to simplify, we do actually have to simplify, not just deprecate or encourage. So if we bring in something new, it must actually cover _all the use-cases_ of what was there before and if we're going to remove choices, there need to be pathways for _all_ the use-cases which were possible with the removed choices. _All_ means **100%** or "without exception"!

And that's kinda what made this such a big project:
- It supports (or at least it's supposed to support!) 100% of the actual ways of using OCaml which exist today.
- It supports (or at least it's supposed to support!) 100% of the platforms that OCaml runs on with these enhancements.
- It doesn't require projects _using_ OCaml to be altered (either code or, more importantly, build) which means that it doesn't _force_ projects, especially libraries, which need to support older versions of OCaml for a while to have to wait or maintain two different mechanisms

Sometimes the most effective way to solve a technical problem is to change the question.

Sometimes, however, you just have to cater for (a lot) of corner-cases...
