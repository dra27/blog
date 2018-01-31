---
layout: post
title: "A pain in the num"
categories: platform
tags: "ocaml opam"
---
# Fixing a packaging error with num on OCaml 4.06.0 system switches

Back in July 2012, the release of OCaml 4.00.0 marked the start of a slow process to remove libraries and tools from the distribution which are not directly related to the compiler and the OCaml Standard Library. For [OCaml 4.06.0](https://github.com/ocaml/ocaml/tree/4.06.0) it was the turn of the [Num](https://github.com/ocaml/num) library. It is now available as the [num package](https://opam.ocaml.org/packages/num) in opam for OCaml 4.06.0 and with a dummy `.0` version which is "installed" for prior versions of the compiler which include the library in the distribution.

It has always been possible to link with the num library using the num package in [ocamlfind](http://projects.camlcity.org/projects/findlib.html), but in order to maintain compatibility with build systems which don't use ocamlfind, the standalone Num library files are still installed to OCaml's `lib` directory, rather than `site-lib/num` as would be more usual for a standalone package.

Very occasionally, I'm allowed to use operating systems other than Windows, and earlier this month I was looking into an opam problem which was sporadically affecting the installation of [Topkg](http://erratique.ch/software/topkg) with "system" compilers in opam. This led to a [bug-fix in opam](https://github.com/ocaml/opam/pull/3164) but along the way I also spotted that the num package was being incorrectly installed for "system" switches.

The problem for system switches, is that OCaml's `lib` directory (often `/usr/lib/ocaml`) is fundamentally not in `OPAMROOT` (usually `~/.opam`). This means that no package should ever write to it which requires special handling for some packages. ocamlfind, for example, has a special script installed for system compilers to workaround the inability to install a helper script directory to OCaml's `lib` directory.

The num packages for versions 1.0 and 1.1 did not contain any handling for system switches and so were incorrectly installing files outside of `OPAMROOT`. There is no workaround for this - it's simply not possible to install the files in the legacy location using opam. So I [tweaked the build system for Num](https://github.com/ocaml/num/pull/6) to provide a "pure" installation via ocamlfind which could be used for system switches and back-ported this to versions 1.0 and 1.1, which was merged in [PR#11207](https://github.com/ocaml/opam-repository/pull/11207).

Since the num package can only be installed on OCaml 4.06.0 or later, it's not yet commonplace to have system switches with OCaml 4.06.0, because distributions quite naturally lag behind. [Homebrew](https://brew.sh) on macOS is one exception, and has had OCaml 4.06.0 packaged since the day it was released. In addition to being one of very few distributions immediately distributing OCaml 4.06.0, Homebrew also has another unfortunate feature: `/usr/local` is user-writeable. Unfortunately, this means that while installations of num to system switches before 19 January on Linux were causing errors, Homebrew users were seeing no error because the opam package was happily installing the files in `/usr/local/lib/ocaml`.

All this means that the correction to the **packaging** of Num looks a [bug to some users](https://github.com/ocaml/opam-repository/issues/11316). Sadly, if your build system assumes that the Num library files will be present in OCaml's `lib` directory and you want to be able to install via opam to system switches, there is no choice but to fix your build system to use ocamlfind.

Much more importantly for any macOS users who have installed the broken num package to their system compiler, there are incorrectly installed files knocking around in `/usr/local/lib/ocaml`. For now, this isn't a problem, but it will be when a new version of ocamlfind is installed or, even worse, a new version of Num. The problem is that ocamlfind's `configure` script will detect the Num files which are incorrectly installed in `/usr/local/lib/ocaml` and will install a `META` file for the num package. When the num opam package is subsequently installed, it will result in an error as it will try to install the ocamlfind num package a second time, which will fail with an error. If Num 1.2 is released before ocamlfind is updated, the situation is worse because 1.2 will install to `~/.opam/system/lib/num` as it should, but ocamlc/ocamlopt will actually use the version found in `/usr/local/lib/ocaml`, as it has higher priority.

All of which means macOS Homebrew users may wish to clean up their system compiler in advance. It's fairly easy to do: it just means deleting `arith_flags.*`, `arith_status.*`, `big_int.*`, `int_misc.*`. `nat.*`, `ratio.*`, `nums.*`, `libnums.*` and `stublibs/dllnums.*` from the directory given by `ocamlc -where`. [PR#11300](https://github.com/ocaml/opam-repository/pull/11300) adds this advice to the num package.

We'll try to be a little more careful when the `graphics` and `str` libraries are eventually removed from the compiler!
