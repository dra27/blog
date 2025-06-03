---
layout: post
title: "Build Meetup - Jane Street London"
categories: misc
tags: "eeg tarides dune ocaml"
---
Stepping into something different today for a [Build Meetup](https://meetup.build/)
hosted by [Tweak](https://moduscreate.com/), [EngFlow](https://www.engflow.com/)
and [Jane Street](https://www.janestreet.com/) at Jane Street's London offices.
I was quite involved with Jbuilder development and early work around Dune 1.0
and some early 2.x work, although it's not a codebase I get to work on much
these days. What was interesting for me, spending a lot of time in GNU make for
the compiler, was to get some first-hand "big picture" experience from the talks
and also a chance to catch-up with various OCaml people who can be remarkably
hard to pin down.

This is more a mish-mash of thoughts and memories from the day than anything
else - talks were being recorded, so I may try to update some of the details
with links to slides, but I don't have them to hand at the moment.

There were six talks (and in fact a bonus one at the end!).

"Transitioning A Large Codebase to Bazel" (Benedikt Würkner, TTTech Auto). A
theme for me started with this talk and continued through others - the day is
about build systems for vast repositories within very large companies, but the
lessons apply just as readily to disparate smaller systems outside in "public
open source". The talk identified phases of moving a huge codebase maintained by
hundreds (or even thousands) of developers. Getting past the envy of being able
to work in an environment where one has an entire full-time on just "the build
system", I particularly focussed on the necessary part of "Convince" -
especially that that needed to be across the board (Management - QA -
**Engineers**), especially as my feeling of online discussions with `dune pkg`
is that somehow we've missed that part. My limited experience talking to people
working on these huge codebases has been that there's often necessarily a huge
focus on _speed_. It was therefore very interesting to me from the "Execute"
phase of doing things for the key advice to be not blocking on speed, and indeed
the statement that "fast can come later, don't block on future things which need
to be changed" (because I personally think that's been massively missed in our
own efforts - I've always prioritise correctness over speed... fast but
sometimes not working is for me only fractionally above broken).

"Integrating bazel/pre-commit" (Matt Clarkson, Arm). Quite a few years ago, I
added pre-commit linting githook for OCaml ([ocaml/ocaml#1148](https://github.com/ocaml/ocaml/pull/1148)).
I find it quite handy, but my impression that there aren't many others who do.
Holy moly, there's a big infrastructure of githooks out there in use in
companies! TIL about [pre-commit.com](https://pre-commit.com/). Integration of
this with Bazel was relevant, if not replicable - I vociferously fight to keep
our lint script in awk not because I'm mad (well...), but because the point is
that the githook has no dependencies. This was a very neat demonstration of work
to allow a hermetic environment for having diverse hooks potentially in a
different version of Python from the project using them being able to be
deployed and updated easily for users (in this case, of course, developers). The
main focus resonates with work that has been ongoing and which I hope to be able
to continue for the compiler - bringing CI as local as possible, ensuring that
the PR is not the first time you discover the problem.

Next up was a talk on Dune advances withinin Jane Street (Andrey Mokhov + ???).
They've made some changes to allow nix to be used to get external dependencies
(`(sysdeps ..)`) stanza. Jane Street of course get to simplify the world a
little (and, given the amount of code, why wouldn't they!!), but interest to
muse how this could be extended out to both multiple-platforms and also to
opam metadata in general (and the overlap with some of our own work on multi-
ecosystem package solving). The other feature demonstrated was peer-to-peer
remote builds. Motivation of this was interesting to me - I've previously
argued that aspects of Windows support get more easily merged by demonstrating
that what's required is actually critical for something else (as have others:
cf. the excellent ["NTFS really isn't that bad"](https://www.youtube.com/watch?v=qbKGw8MQ0i8)).
Remote building always sounds like a nice idea, but hits problems quite quickly
(reproducibility, etc., etc.). Of course, it becomes really critical when that
remote building involves GPUs - i.e. it's become something more important by
wanting to be able to share and schedule hardware, even though the concept of
remote build servers has been being talked about for years. Nice demonstration
of "doing the right thing" as well - the p2p aspect is neat, and while it was
clear they haven't to actually benchmark its being better, I liked the subtext
that it's been done this (slightly more complicated) way _first_ because the
the simpler centralised system look bottlenecky even without evidence 😊

"Measuring & Improving Build Speeds" (Vaibhav Shah, EngFlow). I've been musing
on (non-evil) telemetry and more continuous measuring of build performance (both
package managers and build systems). I guess the niceish takeaway here is that
this affects large companies too... it's not just projects with a small number
of maintainers who end up only looking at build performance regressions when it
gets really bad and then forgetting about it for a few months/years until it
next gets bad!

"What Makes Buck2 Special?" (Neil Mitchell, Meta). I hope the video of this talk
emerges at some point, because it was really great. In particular, this
identified for Buck2 a distinction of having a static dependency graph (Bazel,
make, etc.) versus a fully dynamic dependency graph (Excel, etc.) as being a
spectrum between having a static _dependency_ graph and sections of a dynamic
_action_ graph. For example, in OCaml terms, that explains that `foo.ml`,
`bar.ml` and `baz.ml` make up `awesome.cmxa` (static dependencies), but still
allow the precise dependencies between those ml files to be dynamically
discovered by `ocamldep`. However, that's not just the build system - this is
similar (probably unsurprisingly, but I was briefly surprised, as it hadn't
occurred to me before) for a package manager where it the distinction between
the _dependency graph_ and the _action graph_. In particular, for Buck2 this
can intuitively be the static dependency graph tells you what is strictly needed
(and is largely specified in the build description) but then the action graph
determines things like parallelism - dynamic, but still guided by the static
dependency graph. Which is _exactly_ the package manager model. Wondering how to
apply that to my own musings for dynamic/property-based discovery of external
dependencies for a future version of opam.

"Extending Buck2" (Andreas Herrmann, Tweag). On the downside - the main subject
of this talk is an internship proposal I floated years ago for Dune which never
got anywhere. On the plusside - it works beautifully in Buck2, so it's
validated! The idea is to be able to break through the boundaries of libraries
to increase build parallelism - in other words, instead of compiling `foo.cmxa`,
`bar.cmxa` and `baz.cmxa` in order to link `main-program`, you actually get to
compile _exactly_ the modules which are used in `main-program` and then link it,
potentially _then_ creating those cmxa files in parallel as usable future
artefacts. That's obviously a quite interesting piece of dynamism - in
particular, it means on a build that you might choose to the cmxa files if
nothing has changed, or you might ignore it completely. Crucially, it provides a
more accurate dependency graph - if you change a module in a library which is
not linked in the resulting executable, you can avoid rebuilds. TIL that Haskell
has a build-system like mode where it can discover dependencies and compile more
files as it goes (I have an intern looking at that in OCaml this summer,
although I'm more interested in seeing how easy it is retrofit using algebraic
effects). And - interestingly, given why I'd come along for the day - the
question was asked as to why more compiler authors aren't in the room with
build system authors, because these kinds of optimisations do clearly have to be
done in coordination with the compiler. So I polished my halo a bit!
