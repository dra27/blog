---
layout: post
title: "Retrofitting a build system into a compiler"
categories: platform
tags: "ocaml eeg"
---

Over the summer, [Lucas Ma](https://github.com/lucasma8795) has been
investigating ideas surrounding using effects [in the OCaml compiler itself](https://anil.recoil.org/ideas/effects-scheduling-ocaml-compiler).
He's [blogged some of his discoveries and adventures](https://lucasma8795.github.io/blog/).
The technical core of this work leads towards being able to use the OCaml
compiler as a library on-demand to create a longer-lived "compiler service". Of
itself, that's not at all revolutionary, but it is quite hard to do that with a
30 year old codebase that really was designed for single-shot separate
compilation.

Lucas got to grips pretty swiftly with OCaml's build system, and initially
looked at generalising a core internal part of the compiler called the
`Load_path`. This is used by the compiler for scanning the various "include"
directories for files, principally typing information. For example, if your code
contains a call to `Unix.stat`, then the type checker needs the typing
information for a module called `Unix` which will cause it to request `unix.cmi`
from the `Load_path` and which will then hopefully resolve that to, say,
`~/.opam/switch/lib/ocaml/unix/unix.cmi`.

Effects provide an elegant way of inverting the control for this lookup, as the
program _calling_ the compiler can then change the way these files are looked
up. It also provides the opportunity to "lie" to the compiler about the files
which are actually present, and this was the first thing Lucas started to do
with this change. In particular, it allows us to ignore the dependency graph.
When compiling a module, OCaml requires all the type information that a module
refers to have been compiled beforehand. If you have a module in `bar.ml` with
interface in `bar.mli` and where the code refers to `Foo.value`, then OCaml
requires `foo.mli` and `bar.mli` both to have been compiled before `bar.ml` is
compiled. However, thanks to this effectful trick, Lucas could instead allow the
compiler to start with _just_ `bar.ml`. When `Foo.value` is encountered, there's
a request made for `foo.cmi`, at which point, in the first prototype, the
compiler then quickly spawned another instance of itself to compile `foo.mli`
and _then_ resumed compilation for `bar.ml`, with the same trick then happening
at the end of the compilation with `bar.cmi`. i.e. three files (`foo.mli`,
`bar.mli` and `bar.ml`) all compiled just from `ocamlc -c bar.ml`.

Possibly neat for being able to remove [monstrosities like this](https://github.com/ocaml/ocaml/blob/trunk/.depend)
from OCaml's source tree one day, but so far not _so_ exciting. However, effects
give us more than just hooks into the compiler's operations. We've got an
entire suspended compilation packaged up in a continuation... which means that
that same compiler "process" can now do something else. The next trick was to
have it that instead of spawning a new compiler, the current process itself
returned back into the compiler and itself compiled the required interface file
and _then_ simply resumed the continuation of the previous filke. At this point,
the 30-year-old codebase rears its head again. For reasons of speed and space,
many parts of the compiler, especially in the type checker, feature a lot of
global mutable state. In particular, the compilation pipeline is _not_
re-entrant. Luckily, thanks to the Merlin project, there is a mechanism in the
type-checker for taking snapshots of all this global state. Lucas was able to
piggy-back on this so that, just before the compiler performs an effect to
request a .cmi file (that doesn't yet exist), it snapshots all its global state,
performs the effect and then, when resumed, restores that state again.

Using this to interrupt type-checking and start on something else isn't quite
what this [`Local_store` mechanism](https://github.com/ocaml/ocaml/blob/trunk/utils/local_store.mli)
was originally intended for, and there was a bit of debugging to find a few more
pieces global state which weren't being "registered", but Lucas was able to get
a means of building the OCaml bytecode compiler with nothing pre-compiled where
all the compiler had to be given was the list of .ml files required. From a
toolchain perspective, we're essentially retiring [`ocamldep`](https://ocaml.org/manual/5.3/depend.html).

So far, still mostly just so neat: one single compiler process (just about)
successfully recompiling the compiler. However, that's equivalent to compiling
with `make -j1` - a sequential, and therefore slow, build. The awesome part came
next - Domains. In the final version Lucas was working on, multiple domains were
started up, each one beginning compilation of one of the .ml files required for
the compiler _in parallel_, with a scheduler handling effects coming from each
of these in turn when .mli files needed compiling, and despatching those. The
`Local_store` mechanism in the came in handy here - Lucas extended it to use
[Domain Local Storage](https://ocaml.org/manual/5.3/api/Domain.DLS.html),
combined with the snapshotting. The prototype - for simplicity - featured no
sharing between these domains.

By the end of the summer, this was _very_ nearly working, which is a result
consiserably further than I'd expected in the time available! As is so often the
case with these investigations, Lucas's work had revealed some new facets to
this area that weren't clear to me before. I had previously been wondering how
we would be exposing this kind of multi-threaded compiler to the user via the
driver programs, but it became increasingly clear that this wasn't something
that would be necessary - the program that we were working on to build the
compiler itself was of course not the compiler driver, _but a build system_. To
me, there are two particularly exciting things about that:

1. It's a _really simple_ build system. Hopefully when the last few kinks in the
   parallel type checker are ironed out (read on...), we may be able to add that
   it's really simple **and performant**.
2. It's fundamental portable. It leads to the possibility of bootstrapping OCaml
   trivially with itself. This has been done before with `ocamlbuild`, but the
   result was a maintenance disaster. However, the sheer simplicity of the
   multi-domain effect-scheduling approach is making this perennial build system
   hacker tinker...
