---
layout: post
title: "OCaml 5.4 and opam 2.4 on their way"
categories: platform
tags: "ocaml windows"
---
[opam 2.4](https://opam.ocaml.org/blog/opam-2-4-0-alpha1/) was branched last
week… very pleasing to see [Ryan’s](https://ryan.freumh.org/) work on Nix depext
support get merged (we spent quite a bit of time on that together last summer).
It’s a subtle-sounding (huge) change, but the move away from relying on `patch`
and `diff` as external commands (which has been a HUGE amount of work done by
[@kit-ty-kate](https://github.com/kit-ty-kate)) paves the way for being able to
sort out the incredible slowness of `opam update` on Windows.

[Not at all coincidentally](https://icfp24.sigplan.org/details/ocaml-2024-papers/10/Opam-2-2-and-beyond),
OCaml 5.4 was frozen two days ago as well. Relocatable OCaml not quite ready in
time, but at least those PRs will be ready really, really\[, really\] soon 🫣…
