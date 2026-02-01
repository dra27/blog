---
layout: post
title: "It’s merged!!"
categories: platform
tags: "ocaml relocatable"
---

As I was very happy to [announce on Discuss](https://discuss.ocaml.org/t/volunteers-to-review-the-relocatable-ocaml-work/16667/11)
on 12 December, [OCaml is Relocatable](https://github.com/ocaml/ocaml/commit/7e71861a1ac1ea472104f40168f42133da02096c)!
Today, the final piece of the puzzle was merged, which is the necessary support
to allow opam to take advantage of all this to be able to clone switches instead
of recompiling them. Before this, you could _rename_ a local switch, for
example, but opam would still be compiling them from scratch. The fruits of this
continue to be available [through my opam-repository fork](https://discuss.ocaml.org/t/relocatable-ocaml/17253),
as [announced in September]({% post_url 2025-09-15-relocatable-ocaml %}), and
will be available out-of-the-box for the first alpha release of OCaml 5.5 early
next year.

I am particularly thrilled, and just a little surprised, that Relocatable OCaml
has happened as I envisioned, with nothing needing to be dropped. There were of
course many useful suggestions for re-working to improve clarity and so forth
during review, but the test harnesses, RFCs, talks, and so forth appear to have
provided me with the required knife to work out correctly what was _necessary_
to achieve all this!

For posterity, here’s the timeline! On 4 May, [ocaml/ocaml#14014](https://github.com/ocaml/ocaml/pull/14014)
introduced a comprehensive test harness for the [Relocatable OCaml RFC](https://github.com/ocaml/RFCs/pull/53),
which was reviewed by [Nicolás Ojeda Bär](https://github.com/nojb) and
[Olivier Nicole](https://github.com/OlivierNicole) and merged on 21 June.

Finalising the PRs took me a little while longer than I’d expected, and the set
of 4 PRs, along with a fifth “combined” demonstration [were opened on
15 September](https://github.com/ocaml/ocaml/pulls?q=is%3Apr+author%3Adra27+created%3A2025-09-15).
Over the coming weeks, [Jonah Beckford](https://github.com/jonahbeckford),
[Antonin Décimo](https://github.com/MisterDA), [Hugo Heuzard](https://github.com/hhugo),
[Samuel Hym](https://github.com/shym) and [Vincent Laviron](https://github.com/lthls)
pored over the details. Their invaluable review then set the scene for a
marathon synchronous [defence of the branches](https://bsky.app/profile/dra27.uk/post/3m6htzdhxok2f)
in Paris with [Damien Doligez](https://github.com/damiendoligez) on 25 November.
I’m very grateful to everyone involved in these reviews: a lot of code; and a
lot of gnarly details!

That left a small todo list for each PR. I managed to coerce a different core
maintainer for each, with [Florian Angeletti](https://github.com/Octachron)
merging [ocaml/ocaml#14243](https://github.com/ocaml/ocaml/pull/14243) on
27 November, [Gabriel Scherer](https://github.com/gasche) merging
[ocaml/ocaml#14244](https://github.com/ocaml/ocaml/pull/14244) on 8 December,
Nicolás merging [ocaml/ocaml#14245](https://github.com/ocaml/ocaml/pull/14245)
on 12 December and then finally [Nick Barnes](https://github.com/NickBarnes)
merging [ocaml/ocaml#14246](https://github.com/ocaml/ocaml/pull/14246) today.

That means that OCaml’s trunk branch now matches the bulk of [ocaml/ocaml#14247](https://github.com/ocaml/ocaml/pull/14247).
What’s next? There’s a small amount of work still to do on the scripts which
plumb the ocaml opam package together. Then, having now got everything merged,
it’ll be time to finalise the backports for proposed re-releases of the older
compilers.

But, for now, the future’s very definitely relocatable! 🥳
