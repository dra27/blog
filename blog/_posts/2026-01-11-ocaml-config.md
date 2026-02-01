---
layout: post
title: "Forging compilers in opam"
categories: platform
tags: "ocaml opam relocatable"
---

As we settle into 2026, I have been doing a little early spring-cleaning. A few
years ago, we had a slightly chaotic time in opam-repository over what should
have been a migration from gforge.inria.fr to a new [GitLab instance](https://gitlab.inria.fr/).
Unfortunately, some release archives effectively disappeared from official
locations, and although the content was available elsewhere, the precise
archives weren’t generally available, which is a problem for the checksums in
opam files. We’ve had [similar problems](https://github.com/ocaml/opam-repository/pull/23849)
[with GitHub](https://github.com/ocaml/opam-repository/pull/10182) in the past.
As a ‘temporary solution’, [@avsm](https://github.com/avsm) created [ocaml/opam-source-archives](https://github.com/ocaml/opam-source-archives/commit/bad1a54d3e64267a3e5b5e9e13083d55cc176307)
to house copies of these archives (I think it’s a somewhat prescient sha for
that first commit!). As is so often the case with temporary solutions, it’s
grown somewhat. Rather against my personal better judgement, the repo got used
to house files which used to be shipped as part of [ocaml/opam-repository](https://github.com/ocaml/opam-repository).
Removing the files from the repository was a good change, because they were
always being shipped as part of `opam update`, but unfortunately moving them to
an “archive” repository has made it rather too tempting to add _new_ files,
making an archive repository a primary source.

Back in September, as part of Relocatable OCaml, I needed to update the
ocaml-config package, which houses one of the plumbing scripts used for the
[`ocaml`](https://opam.ocaml.org/packages/opam) package in opam. Separately, for
opam’s CI systems, we wanted to be able to test against trunk OCaml, which
implied [some updates to the plumbing](https://github.com/ocaml/opam-repository/issues/23515)
for [`ocaml-system`](https://opam.ocaml.org/packages/ocaml-system) as well. The
right thing to do with these scripts which had lived in opam-respository is to
push them back upstream, which was what I did with a cute piece of Git
spelunking in [ocaml/ocaml#14351](https://github.com/ocaml/ocaml/pull/14351/commits),
which contains commits with files cherry-picked from [ocaml/opam-repository](https://github.com/ocaml/opam-repository)
PRs. Each commit contains a reference to an opam-repository commit, which in
turn leads to the original PR. For example, [ocaml/ocaml#d0272f8](https://github.com/ocaml/ocaml/commit/d0272f845e90f8280e821f90382b3c189af00ea6)
copies the original files from [ocaml/opam-repository#1bab453](https://github.com/ocaml/opam-repository/commit/1bab4537a2f3b2328d771e1bb50b0d0268b0798a).
The neat thing with putting them into a series under a single path in OCaml now
is what then happens with subsequent changes. For example, [ocaml/opam-repository#17541](https://github.com/ocaml/opam-repository/pull/17541)
introduced the `ocaml-config.2` package, which was a completely fresh script,
but [ocaml/ocaml#749a918](https://github.com/ocaml/ocaml/commit/749a918bb85033b0a6370b85a7c6a4be33620c58)
is instead able to show the actual diff of the script, allowing for much easier
review and so forth. Of course, git doesn’t store patches, so the really useful
part is that although the history is different, the _file_ in each commit is
exactly as in the original commit, which allowed [ocaml/opam-repository#29080](https://github.com/ocaml/opam-repository/pull/29080)
just to update the URLs to point to these _authoritative_ upstream sources.

So far, so good - that had all been merged before Christmas. The support for
explicit-relative paths in `ld.conf` added in [ocaml/ocaml#14243](https://github.com/ocaml/ocaml/pull/14243)
required an update to this script, as noted in [ocaml/opam-repository#29085](https://github.com/ocaml/opam-repository/pull/29085),
since `opam var ocaml:stubsdir` for OCaml 5.5 and onwards was giving an
erroneous `../stublibs:./stublibs:.` as it didn’t know to translate the paths
read from `ld.conf` as being relative to `ld.conf` itself. That an update was
required anyway gave me opportunity to fix three other oddities in that script:

1. The script gets installed to users’ opam switches, rather than used directly
   as part of the build.
2. There were several versions of it, and there didn’t need to be.
3. The use of opam’s `substs` mechanism made it difficult to debug outside of
   opam.

This work had all been left in [ocaml/ocaml#14250](https://github.com/ocaml/ocaml/pull/14250)
[in December](https://github.com/ocaml/ocaml/commits/78ca20415620060c279ed74a0ea5b22c06d7e548)
and was straightforward enough. The ocaml-config package had only come into
existence in [ocaml/opam-repository#11928](https://github.com/ocaml/opam-repository/pull/11928)
in order to stop the same script being stored multiple times in the same
repository. Once [ocaml/opam-repository#25960](https://github.com/ocaml/opam-repository/pull/25960)
reduced that single copy to zero copies, it made more sense for each ocaml
package just to download the script download, and do away with the ocaml-config
package completely. Unifying the scripts, and turning it into a regular OCaml
script flows logically from that. Although, the final lines amuse me somewhat:

```ocaml
let oc = open_out package_config_file in
  (* Quoted strings need OCaml 4.02; "\ " needs OCaml 3.09! *)
  Printf.fprintf oc "\
    opam-version: \"2.0\"\n\
    variables {\n  \
```

This week, I updated that commit series to do the same kind of thing to the
ocaml-system script. These are now just regular OCaml scripts (in [tools/opam](https://github.com/ocaml/ocaml/tree/trunk/tools/opam)
in [ocaml/ocaml](https://github.com/ocaml/ocaml)) which can be run directly and
that allowed me to update [ocaml/ocaml#14354](https://github.com/ocaml/ocaml/pull/14354)
which should allow us to be offer an ocaml-system package _during_ development
and release cycles. The autoconf stuff might get revisited in that: cute, but
potentially annoying (the joy of higher order meta-programming… autoconf
generates and updates those files as part of the process of generating
`configure`, rather than as part of running `configure` itself… what could
possibly go wrong!). There’s also [ocaml/ocaml#14355](https://github.com/ocaml/ocaml/pull/14355)
which allows a similar trick to be done with custom compilers. The point here is
that you build a non-standard compiler and install it outside of opam and this
then provides a relatively simple mechanism for opam to be able to use it.
Although “system” compilers are a bit awkward to use from distributions,
because opam interacts very badly with system-installed OCaml packages, if the
_only_ thing you’re installing at a system-level is the compiler, the support is
very good.

The final thing to deal with was a very silly ToDo item I’d added:

```
- [ ] `tools/opam/Layout.md`
```

In the excitement of getting the other Relocatable OCaml PRs opened in
September, in a brief from sanity, I had decided some kind of documentation
ought to be in place before [ocaml/ocaml#14250](https://github.com/ocaml/ocaml/pull/14250)
was considered for merging 😱 Days turned into weeks, weeks turned into months.
And then, one not-so-very special day, I went to my laptop, I sat down, and I
wrote our compiler packaging story. A story about package names, a story about
compiler variants, a story about ancient libraries, long forgotten. But above
all things, a story about OCaml.

Which is mercifully now merged.
