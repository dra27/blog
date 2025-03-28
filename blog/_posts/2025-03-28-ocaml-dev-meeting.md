---
layout: post
title: "OCaml Core Dev Meeting"
categories: platform
tags: "ocaml windows"
---
OCaml Core Dev meeting at [Inria](https://inria.fr) yesterday. These are
roughly biannual synchronous catchups which provide a chance to find out what
others are up to, get feedback on any major ongoing work, and attempt to unblock
some stalled PRs. Sometimes tempers get frayed, but not this time round...

Quick round-table: ongoing work to get modular explicits/implicits into OCaml; a
lot of work on GC pacing to get the 5.x GC to follow its space overhead setting
properly; lot of work on flambda 2; OCaml Language committee has been launched
and has two proposals under consideration; quite a lot of reviewing of new
features from Jane Street in flight; work on pluggable GC to be able to try out
alternate GCs in the main runtime.

Two presentations on the table. The OCaml 5.x GC isn't working hard enough to
keep to its space overhead - Jane Street had seen issues with it, as have
Semgrep (and [blogged about it](https://semgrep.dev/blog/2025/upgrading-semgrep-from-ocaml-4-to-ocaml-5/)).
[Stephen Dolan](https://github.com/stedolan), [Nick Barnes](https://github.com/NickBarnes)
and [Damien Doligez](https://github.com/damiendoligez) have been doing a lot of
investigation and work recently into this, which Stephen presented. Lots of
maths and logical argument leading to a very pleasingly small patch which vastly
reduces the overshoot of the 5.x GC. Unlikely to be in 5.4, but it's in use
already inside JS and should be with us for 5.5.

My Relocatable OCaml was next, with fewer graphs, but a functioning demo!
Managed to push the [RFC for it](https://github.com/ocaml/RFCs/pull/53) a few
hours before the meeting. The combined version of it passed [on all platforms](https://ci.inria.fr/ocaml/job/precheck/1030/)
in the early hours of Thursday morning! Main aim presenting it's to get sign-off
on the principles behind it, see if anyone's too horrified by the suggested
design, and then beg to see if it can still be being reviewed past the feature
freeze for 5.4. Well-received - a few bits of useful feedback to experiment with
and as ever getting these things reviewed will be fun.

Feature freeze for OCaml 5.4 on 15 April - useful discussion on reviewing
requirements for large changes coming out of Jane Street (and affecting Tarides
a bit, too).

Presentation and considerable discussion on pure functors ([ocaml/ocaml#13905](https://github.com/ocaml/ocaml/pull/13905)).
Didn't particularly reach a conclusion.

Follow-up discussion on [ocaml/ocaml#12307](https://github.com/ocaml/ocaml/pull/12307),
looking to stop using MD5 in the Standard Library (finally). It's amazing how
backwards-compatibility concerns end up eating so much time with these things:
plan is to switch the default hashing to 128bit Blake2 in 5.5 (which yields
hashes of the same length as MD5 as a stop-gap to check how bad any breakage is)
and then perhaps switch it again to something stronger. Main aim is to ensure
that we get the ecosystem to a point where anyone using a _default_ hash really
isn't relying on the choice of what that is (and it means we stop getting
criticised for the consistency checksums in OCaml's .cmi files and so forth not
being a crytographically secure hash function.......).

Some more maintenance discussions: we'd vastly reduced the complexity behind
building the `Dynlink` library in 5.3, and I'd had a quick sprint one weekend in
January on removing loads of duplicated code in the toplevel by shifting
`Dynlink` into the Standard Library ([ocaml/ocaml#13745](https://github.com/ocaml/ocaml/pull/13745)).
Decision instead of moving `Dynlink` towards the Standard Library was to try
moving the toplevel further out in the build, making its build more like the
debugger. Sewing kit needed on the PR, but the code-reduction effect will still
be the same! Also a quick "temperature in the room" discussion on whether it'd
be a good idea at some point to split the parts of the Standard Library which
are really about the runtime into a separate library (the various "internal"
modules, `Obj`, `Marshal`, `Callback`, etc.). Surprisingly diverse reasons for
possibly wanting to do that, ranging from making the GC clearer to understand in
the runtime, to helping with Standard Library replacements (Core, etc.).
Maintenance discussion on `ocamltest` as well - conclusion is that there's quite
a lot of things we'd like to do to improve it, and some of us may even get round
to doing some of them! 32-bit platforms were talked about, too - what might we
gain if we stopped supporting them. There's various bits of code in the runtime
which some of us might wish weren't there (but which aren't necessarily causing
maintenance pain). There was some hope from the Jane Street side that we might
be able to get rid of the need to track immediate64 (that is, values which are
immediates on a 64bit system but are boxed on a 32bit system), but both
JavaScript and WebAssembly mean that even if we stopped building the runtime on
32bit systems, we'd still need that that distinction internally. Conclusion for
now is that the bell is probably tolling, but it's not yet time to remove them
(which keeps me happy for now... there's a hoarder in me that never really wants
to remove anything which still actually works!).

Atomic records ([ocaml/ocaml#13404](https://github.com/ocaml/ocaml/pull/13404)
and [ocaml/ocaml#13707](https://github.com/ocaml/ocaml/pull/13707)) got
discussed again. Just about agreed that `[%atomic.loc]` might go in - but
there's still a lingering concern that pointers might creep in...! More naming
things, too (why is naming so hard?!) - although in this case we've accidentally
ended up with a proposal for a function intended to block on a mutex having
`_non_blocking` in its name, so maybe naming really is that hard! Similarly, an
agreement that having `let+`, `and+`, `let*` and `and*` operators in submodules
should go ahead (this is for `Result.Syntax` in [ocaml/ocaml#13696](https://github.com/ocaml/ocaml/pull/13696)),
but possibly with even more discussion on the name `Syntax` required 😁

Following on from the last meeting in October, there'd been some ideas behind
being able to signal to the GC that a program is not in the "steady state"
(where memory is allocated at the same rate as memory is becoming garbage).
That's been exposed as two new functions `Gc.ramp_up` and `Gc.ramp_down` in
[ocaml/ocaml#13861](https://github.com/ocaml/ocaml/pull/13861) and benchmarking
in Rocq is showing that it's beneficial when you have a lot of data to
unmarshal. Sounds like that'll be in 5.4 - the same workloads may also benefit
from the GC pacing changes, but having programs be able to tell the GC that
they're either in the process of allocating or releasing large amounts of memory
is useful knowledge, regardless.

The evaluation order is not defined! Except that programs seem to rely on it. It
turns out that having it change in subtle ways in the compiler is confusing and
problematic for implementation reasons, too - we agreed that it should be made
consistent **if still technically not defined** ([ocaml/ocaml#13882](https://github.com/ocaml/ocaml/pull/13882)).

We don't have enough ways to write strings in OCaml sources ([ocaml/ocaml#13860](https://github.com/ocaml/ocaml/issues/13860)),
but given that we'd end up using this "readable multi-line quoted string
literals" in the compiler's sources, if we added them, there's a consensus that
we should come to a consensus on how to write them!

Finally, as we steamed towards the end of the meeting, we agreed that this:

```
type t = T
type u = t
type v = u = T
```

identified in [ocaml/ocaml#13872](https://github.com/ocaml/ocaml/issues/13872)
would still not be allowed, but thank goodness that "Their kinds differ" is no
longer the error message.

Not quite as many things settled on for stalled PRs and so forth as we did in
October's meeting, but various things moved forward. Now to continue steaming
towards actually opening the pull requests for Relocatable OCaml...
