---
layout: post
title: "Opening up old release branches"
categories: platform
tags: "ocaml relocatable"
---

The spring-cleaning continues! When I originally prototyped Relocatable OCaml,
it was during the OCaml 4.13 development cycle. The focus for the work
originally was always about multiple versions of the compiler co-existing
without interfering with each other, so even the early prototypes were done on
both OCaml 4.12 and OCaml 4.13. In fact, I see that in [this talk](https://discuss.ocaml.org/t/ocaml-cafe-wed-oct-13-1pm-u-s-central/8610),
[I even demo’d it on both versions](https://youtu.be/RHSdlH4el0g?t=1969). My
intention from the start had always been to be able to provide either backports
or re-releases of older compilers, on the basis that it would be tedious to have
only the latest releases of OCaml supporting the various fixes, given that the
failing CI systems which had motivated the project would continue to test older
versions for several/many years after completion. In 2021, OCaml 4.08 (from
June 2019) was still a recent memory. From a technical perspective, OCaml 4.08
was a very important release. It’s the first version of OCaml with a reliably
namespaced Standard Library (the `Stdlib` module, though introduced in 4.07, had
various issues with shadowing modules which weren’t completely addressed until
4.08). For my work, it was the version where we switched the configuration
system to autoconf, and thus introduced a configuration system for the Windows
ports. It provided a natural baseline in 2022 for the backports, and thus the
workshop demonstration I gave in Ljubljana featured Windows and Linux for OCaml
4.08-4.14 as well as preview of OCaml 5.0.

Wind forwards 3¼ years, and OCaml 4.08 is certainly a distant memory, but the
features in it means it is very much a baseline at the moment. So, partly
because the work is mostly already done, partly because it’s the last piece of
the Relocatable puzzle, and mainly because I think it will be useful, I’ve been
working towards proposed branches to allow us to make some exceptional releases
in these old branches.

OCaml 5.x means that we have already been maintaining OCaml 4.14 as an LTS
release of OCaml (although the changes coming in OCaml 5.5 mean that we really
will be looking to end that long-term support before too much longer). I’ve
previously talked about [synchronising the GitHub Actions workflows]({% post_url 2025-05-24-wtw-21 %}#gha).
and [the crazy shell script that manages the backporting]({% post_url 2025-07-29-taming-buildkit %}#stack).

So, how do you maintain some 20-or-so in-flight PR branches being occasionally
rebased _forwards_ onto updated versions of OCaml and simultaneously
_backported_ on to, initially, 7 compilers and, by the end, 12 compilers? The
answer is: carefully, and with a lot of automation! ‘Carefully’ entailed setting
some important ground rules: for example, the golden rule was that either the
work was rebased _or_ it was being amended. Never both simultaneously. This rule
meant that while effort may be required to rebase _forwards_, the backports
remained fixed, which allowed the patch resolutions to re-settle. In order to
ease the automation, it also occasionally required over-backporting, i.e. taking
some changes back to old compilers which were not strictly necessary, but which
meant that the main branches did not need to be extensively re-written.

Anyway, included in this back-porting was enough work to be able to get CI
running, since once we’ve finished support for a given version, the branch is
mostly left to gather dust.

![Opening up old release branches]({{ site.url }}/assets/2026-01-16/2026-01-16-covered-furniture.jpg){: width="75%" height="75%" }

_Image from [Pinterest](https://uk.pinterest.com/pin/828732768911599733/)_.

Like getting a stately home ready for visitors, the task I spent this week
accomplishing was removing the dust sheets from the CI configurations on these
old branches (4.08-4.13) so that the final backported changes could be properly
tested. The stacks for Relocatable OCaml had these patches ordered as:

1. Unify CI workflows (affecting 4.08-4.13). We started switching over to GitHub
   Actions during the 4.12 dev cycle, but only completed the migration during
   the 4.13 dev cycle, so there were long-dead Travis CI scripts in the
   4.08-4.12 branches, and the 4.13 branch’s GitHub Actions scripts hadn’t of
   course received the updates which the 4.14 ones had.
2. macOS arm64 support (4.08, 4.09 and 4.11). The curious gap is because Apple
   released the M1 during the 4.11 dev cycle, and it was decided that it was too
   late to alter the 4.11.0 release, so a re-release of 4.10 was done with Apple
   Silicon support, and then 4.12.0 “naturally” had the support, as the version
   of trunk to which it had originally been merged. As it happens, the 4.12
   patches apply completely trivially to 4.11 and the 4.10 version of the
   patches apply trivially to 4.08 and 4.09, which means that you can get the
   complete support matrix when working on Apple Silicon.
3. Windows FlexDLL bootstrap overhaul (affecting 4.08-4.12). The main thing here
   is [ocaml/ocaml#10135](https://github.com/ocaml/ocaml/pull/10135), which made
   the ability to build the Windows ports of OCaml somewhat easier. The lack of
   this branch is the reason why the support for Windows in opam-repository at
   the moment from [ocaml/opam-repository#25861](https://github.com/ocaml/opam-repository/pull/25861)
   only goes back as far OCaml 4.13. It’s not because it’s impossible, it just
   requires a lot of hardening in the opam packages to work around various
   deficiencies, and it’s actually just a configuration/build system change
   which backports really very easily as far as 4.08 (getting 4.07 and earlier
   working in opam on Windows is a [daftness for another day](https://github.com/metastack/ocaml-legacy)).
4. Required backports for Relocatable OCaml (actual changes in the
   runtime/compiler which must be present to support the main PRs).
5. runtime-launch-info backport (from OCaml 5.2; [ocaml/ocaml#12751](https://github.com/ocaml/ocaml#12751))
6. compiled-primitives backport (from OCaml 5.3; [ocaml/ocaml#12896](https://github.com/ocaml/ocaml#12893))
7. in-prefix-test harness (from OCaml 5.5; [ocaml/ocaml#14014](https://github.com/ocaml/ocaml#14014))

Number 1 is exactly where I wanted it, but unfortunately the test harness sits
on top of a large number of other changes, mainly because it was actually
written well after all of them! So I got out my Git [hammer](https://en.wikipedia.org/wiki/Mj%C3%B6lnir)
and began to re-forge the commit series. It’s quite a set, but at the end of it,
the difference between 4.08 and 4.14 in `appveyor.yml`, `.github/workflows` and
`tools/ci` is a small set of trivially justifiable changes relating to actual
differences between those two releases (for example, OCaml 4.08 still had the
graphics library to test!). Ignoring changes that affect the build, configure
and test systems only, it requires just three PRs to resurrect that branch:
- [ocaml/ocaml#9557](https://github.com/ocaml/ocaml/pull/9557) - a necessary
  compilation change on IBM POWER
- [ocaml/ocaml#9981](https://github.com/ocaml/ocaml/pull/9981) - a necessary
  compilation change for the BSDs
- [ocaml/ocaml#12577](https://github.com/ocaml/ocaml/pull/12577) - a series of
  C17 compatibility changes needed for compilation with recent GCC/Clang

… and that’s it!

On top of that unified CI, it’s then possible to backport the test harness,
which then provides reliable installation testing for all the relocatable
features. I’m almost certainly strange, but this was kind of fun to backport. By
the time it gets back to 4.08, the commit message for the harness itself reads:

```
Deviations from original:
  - C stub added to the harness instead of backporting
    caml_sys_proc_self_exe
  - Compmisc.reinit_path used instead of updating Compmisc.init_path
  - Config.c_compiler_vendor avoided by instead exposing the constant as
    Toolchain.c_compiler_vendor
  - Config.asm_dwarf_version folded into toolchain.ml.in
  - Config.shebangscripts avoided by instead adding an additional
    --with- option to the harness and threading $(SHEBANGSCRIPTS) from
    the build system
  - Required parts of Bytelink.read_runtime_launch_info directly added
    to the test harness, instead of exposing the machinery in Bytelink.
  - Misc.Stdlib.String.to_utf_8_seq added directly to TestRelocation.
  - ocamlmklib -M exits with code 0
  - Standard Library functions added:
    - Bytes.{get_utf_8_uchar,set_utf_16le_uchar} (4.14)
    - Char.Ascii.is_letter (5.4)
    - In_channel.{fold,input}_lines (5.1)
    - In_channel.input_all (4.14)
    - In_channel.really_input_bigarray (5.2)
    - In_channel.{set_binary_mode,with_open_bin} (4.14)
    - Int.max (4.13)
    - List.{drop,take}_while (5.3)
    - List.find_map (4.10)
    - List.fold_left_map (4.11)
    - Option.exists (5.5)
    - Out_channel.{with_open_bin,with_open_text} (4.14)
    - Result.Syntax (5.4)
    - String.{starts,ends}_with (4.13)
    - Sys.{mkdir,rmdir} (4.12)
    - Sys.signal_to_string (5.4)
    - Uchar.utf_{decode_uchar,decode_length,16_byte_length} (4.14)
    - Unix.realpath (4.13)
  - OCaml 5.4 labelled tuples removed
  - Added the ability for Build/Prefix/Relative to be optional in
    TestRelocation
  - Prior to 5.4.0, ocamlrun -config didn't include entries in the
    search path coming from CAML_LD_LIBRARY_PATH
  - Prior to 5.3.0, the compiler added +flexdll to the search path even
    when -nostdlib is specified, which affects the relocation test when
    bootstrapping FlexDLL.
  - Prior to 5.3.0, ocamlcmt and ocamlobjinfo didn't support -vnum. The
    simplest thing in this instance is simply to skip the test of their
    binaries.
  - Prior to 5.3.0, the dynlink library included a complete copy of the
    Config module, which affects the relocation test.
  - Prior to 5.2.0, RNTM isn't always written to bytecode images when
    compiling with -custom. Heuristic altered to analyse the DLLS
    section instead.
  - Prior to 4.14.0, errors in #load don't create detectable script
    errors: toplevel test adapted to cope with this
  - Prior to 4.13.0, the MSVC port didn't translate -lfoo automatically
  - Adapt the toplevel test to deal with the presence of compiler
    plugins in ocamlnat
  - Link test updated to deal with different runtime behaviour for
    errors and the lack of -output-complete-exe, both of which changed
    in 4.10.0
  - Build system adapted (Makefile.common macros didn't support C files
    until OCaml 5.2; VPATH wasn't used until OCaml 5.1; win32unix/unix
    distinction until OCaml 5.0)
  - Misc.Style changed to Misc.Color (changed in OCaml 5.2) and uses of
    the inline_code and hint formatting changed to loc (since
    inline_code was also added in OCaml 5.2 and hint OCaml 5.1)
  - Removed compressed marshalling support, since that was added in
    OCaml 5.1
  - Read camlheader file instead of runtime-launch-info, and adapt the
    relocation test to deal with incorrect naming of two of the header
    files (fixed in 4.09.0)
  - OCAML_FLEXLINK scrubbed from the environment (removed in 5.2)
  - Updated to deal with all bytecode binaries being installed with
    debug information
  - Updated to deal with all bytecode/native versions of tools being
    installed (changed in 5.1)
  - Updated to deal with old installation layout for otherlibs
  - Updated handling for the bigarray library
  - Updated relocation test to deal with different use of debug flags
    from OCaml 4.12+
  - Order of rules in testsuite/in_prefix/Makefile.test updated for
    compatibility with the aged version of GNU make shipped on macOS
    (pattern rules triggered in the wrong order otherwise)
```

Next-up, having removed all those dust sheets: finalising the actual backports!
