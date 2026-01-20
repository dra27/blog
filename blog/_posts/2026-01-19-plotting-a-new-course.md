---
layout: post
title: "open Core"
categories: platform
tags: "ocaml oxcaml"
---
On 16 December 2000, a young @dra[^1] stepped out on to the stage of [St Martin-in-the-Fields](https://www.stmartin-in-the-fields.org/)
making what would be the first of many performances of Johann Sebastian Bach’s
great [Mass in B minor](https://en.wikipedia.org/wiki/Mass_in_B_minor). On 16
November last year, just under 25 years later, a slightly greyer @dra27 stepped
out on the stage of [King’s Hall](https://www.newcastlebachchoir.org.uk/dbpage.php?pg=view&dbase=events&id=202182)
at Newcastle University for what, for now at least[^2], would be his last
performance of this great work[^3]. As I write this in the 9 hour window of
unemployment between finishing at the [University of Cambridge](https://www.cam.ac.uk)
and [Tarides](https://tarides.com) and commuting down to 2½ Devonshire Square to
start at [Jane Street](https://www.janestreet.com), it’s a new year and a change
of course.

[^1]: the “27” wouldn’t be allocated until the next October
[^2]: never say never…
[^3]: I’ve never recorded the work, although I recorded [Ach, bleibe doch](https://open.spotify.com/track/7kZypUPNwPu0JKCCbD57X2) from [Himmelfahrtsoratorium BWV 11](https://en.wikipedia.org/wiki/Lobet_Gott_in_seinen_Reichen,_BWV_11#4) with [Musik Podium Stuttgart](https://musikpodium.de) ten years ago, which is one of the source arias for the famous _Agnus Dei_ of the mass

My professional life to now has always been a balancing act between the arts and
technology (perhaps a rollercoaster would be a better analogy; balancing act
somehow evokes the elegance and skill of a trapeze artist). I’ve [mused before]({% post_url 2025-07-18-through-a-glass-darkly %})
on some of the common threads that drive me; more recently I’ve been musing on
more fundamental similarities. Many years ago, I remember in some Cathedral or
other being told of the various carvings which exist in hidden parts of these
buildings; art created not to be seen, at least by human eyes. Amongst others,
restorations at [Salisbury Cathedral](https://www.salisburycathedral.org.uk)
uncovered [such carvings](https://www.theguardian.com/commentisfree/2023/sep/12/the-guardian-view-on-the-hidden-carvings-of-salisbury-cathedral-messages-to-the-future).
Effort expended not for human reward, but for its own worth or, one could say,
[ad maiorem Dei gloriam](https://en.wikipedia.org/wiki/Ad_maiorem_Dei_gloriam).
Or, in technology, The Right Thing™. The right thing is what instantly drew me
to functional programming back way before it was cool and shortly after the “27”
had been added to “dra”. The pragmatic approach of OCaml trying to balance the
safety, correctness, and Right Thing of functional programming with the need to
write performant programs in a less Right Thing-like world made it a natural
choice for a young professional singer writing and maintaining small systems
written on trains, planes and hotels around the world! It’s continued to draw me
in over the last 9 years.

But for me the art _is_ made to be seen. During the COVID-19 pandemic, when live
performance became impossible, I remember spending many months at home unable,
or at least unwilling, to sing. Without even the [colleagues to perform with](https://open.spotify.com/track/6nQeKKpvUFJE0S6gwem2JN),
let alone the audience to consume the result, there was no purpose. And so too
the perfect software is without purpose without users[^4]. In championing and
furthering Windows OCaml, I chose the niche of a niche, but I am hugely proud
that today _every_ Windows user of OCaml benefits (hopefully!) from the work I
[both](https://github.com/ocaml-multicore/ocaml-multicore/pull/351) [did](https://github.com/ocaml/ocaml/pull/11642),
and [spearheaded others to do too](https://github.com/ocaml/ocaml/pull/12954).
Likewise, _every_ Windows user running `winget install opam` begins their
journey in OCaml following my vision of how [it should work](https://github.com/ocaml/opam/issues/246#issuecomment-2166133625),
thanks to the seemingly boundless patience and efforts of my opam
co-maintainers!

[^4]: Perhaps I should adopt _sine usoribus sine proposito_ as a motto

Behind all this, though, are the companies which allowed this to happen: first
at OCaml Labs at the University of Cambridge and then spinning out into Tarides.
And behind all that is Jane Street. After I started at OCaml Labs back in 2016,
I explained to (mainly musical) colleagues that I was carrying on doing the open
source work I’d been doing for the previous 10 years, but that somehow that had
become work one could be _paid_ to do (I can’t underscore enough how
inconceivable the idea of that would have felt in 2006). That inevitably led to
the question “so what do they get out of it?” - and the inevitable surprise that
the answer was, directly at least, nothing. Windows opam represented work on a
platform with no business case on an unused tool. The OCaml community was - is -
the benefit.

Which makes this week feel less a change of course and more a continuation.
[OxCaml](https://oxcaml.org) represents to me the evolution of the pragmatism
that drew me to OCaml in the first place. And we’ve definitely got users!
This year, like last year, is clearly going to be [interesting](https://en.wikipedia.org/wiki/May_you_live_in_interesting_times).
I think as we continue to navigate the whirlwind of agentic engineering, we’re
going to care more and more about the foundation it’s all sat on, especially the
compilers. Let’s hope they continue to strive to do the Right Thing™. And let’s
see how this looks next year! So, here goes…

```ocaml
open Core
```

![The cat’s out of the bag]({{ site.url }}/assets/2026-01-19/2026-01-19-tori-from-a-bag.jpg)
