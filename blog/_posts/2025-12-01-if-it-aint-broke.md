---
layout: post
title: "If it ain’t broke, …"
categories: platform
tags: "ocaml windows"
---
Someone was kind enough a few weeks ago to comment on [Hacker News](https://news.ycombinator.com/item?id=45855726)
that “Opam on Windows is a masterpiece of engineering”. There’s certainly a lot
which has to go on under the hood to create what I believe is the necessary
baseline experience. Unfortunately, it doesn’t _always_ work out.

10 days ago, a new user popped up on our [Discourse forum](https://discuss.ocaml.org/t/when-following-install-instructions-for-windows-topkg-and-ocp-indent-fail-to-compile/17514)
with a failure of some core packages to work on Windows. The OP was using MSYS2,
which at the moment has slightly less support than the recommended Cygwin, but
it looked to me like either my [ocaml/opam-repository#28897](https://github.com/ocaml/opam-repository/pull/28897)
or [ocaml/ocamlfind#112](https://github.com/ocaml/ocamlfind/pull/112) ought to
fix things.

Except it didn’t, and it wasn’t until today that I was able to reproduce the
problem, and figure out what was going on. As it happens, once I could reproduce
it, this was easy for _me_ to work out, but it shines an interesting light on to
a subtle bit of opam's Windows internals and consequently something that either
the OP or a packager has misunderstood and it also finally goads me into
standing on my soapbox and shouting from the rafters:

**Code should be changed if and only if a bug is being fixed or an actual
user-facing feature is being implemented! The risk of regression ALWAYS
outweighs the benefit of refactoring UNLESS you're actually changing
something else.**

The target of my particular ire is the wonderful [ShellCheck](https://www.shellcheck.net/)
tool. If you don’t use it already, then:

1. You don’t have to write shell scripts, and should continue not having to
   write shell scripts for as long as possible! However, you may have to one
   day, so:
2. You should absolutely start using it _on the next shell script you have to
   write_. You should also use it to check changes you make to existing scripts,
   too.

But what I’m finally going to come out and just say, is that you should _never_
**ever** change a shell script just to please ShellCheck unless you actually
have a bug report. If ShellCheck grumbles and warns something _might_ cause a
bug, then take it as a challenge - figure out the scenario in which that could
be triggered, test it, fix it. If, as is often the case, it’s warning about an
unhygienic construct which _could_ go wrong but actually never will because, for
example, the file names concerned don’t contain spaces and never will, then
leave that script alone!

What was actually going on here? The OP was seeing this effect when trying to
install [topkg](https://ocaml.org/p/topkg/latest):

```
C:\Users\DRA>opam install topkg
The following actions will be performed:
=== install 1 package
  ∗ topkg      1.1.1

<><> Processing actions <><><><><><><><><><><><><><><><><><><><><><><><><><>  🐫
⬇ retrieved topkg.1.1.1  (https://opam.ocaml.org/cache)
[ERROR] The compilation of topkg.1.1.1 failed at "ocaml pkg/pkg.ml build --pkg-name topkg --dev-pkg false".

#=== ERROR while compiling topkg.1.1.1 ========================================#
# context     2.5.0 | win32/x86_64 | ocaml.4.14.2 | https://opam.ocaml.org#9bc1691da5d727ede095f83c019b92087a7da33e
# path        C:\Devel\Roots\msys2-testing-native\default\.opam-switch\build\topkg.1.1.1
# command     C:\Devel\Roots\msys2-testing-native\default\bin\ocaml.exe pkg/pkg.ml build --pkg-name topkg --dev-pkg false
# exit-code   125
# env-file    C:\Devel\Roots\msys2-testing-native\log\topkg-8308-179b77.env
# output-file C:\Devel\Roots\msys2-testing-native\log\topkg-8308-179b77.out
### output ###
# Exception: Fl_package_base.No_such_package ("findlib", "").
```

This error gets readily seen when using OCaml 5.x on MSYS2. It’s not a problem
with topkg, it’s actually that the [ocamlfind](https://ocaml.org/p/ocamlfind/latest)
library manager is not installed correctly. There are fixes pending for that in
[ocaml/ocamlfind#112](https://github.com/ocaml/ocamlfind/pull/112), but the
issue there is to do with OCaml **5.x** - this build is OCaml 4.14.2.

The actual issue is readily apparent if we rebuild ocamlfind:

```
C:\Users\DRA>opam reinstall ocamlfind -v
The following actions will be performed:
=== recompile 1 package
  ↻ ocamlfind 1.9.8

<><> Processing actions <><><><><><><><><><><><><><><><><><><><><><><><><><>  🐫
⬇ retrieved ocamlfind.1.9.8  (cached)
+ C:\Devel\Roots\msys2-testing-native\default\.opam-switch\build\ocamlfind.1.9.8\./configure "-bindir" "C:\\Devel\\Roots\\msys2-testing-native\\default\\bin" "-sitelib" "C:\\Devel\\Roots\\msys2-testing-native\\default\\lib" "-mandir" "C:\\Devel\\Roots\\msys2-testing-native\\default\\man" "-config" "C:\\Devel\\Roots\\msys2-testing-native\\default\\lib/findlib.conf" "-no-custom" "-no-camlp4" (CWD=C:\Devel\Roots\msys2-testing-native\default\.opam-switch\build\ocamlfind.1.9.8)
- Welcome to findlib version 1.9.8
- Configuring core...
<snip snip snip>
- Configuration for threads written to site-lib-src/threads/META
- Configuration for str written to site-lib-src/str/META
- Configuration for bytes written to site-lib-src/bytes/META
- Access denied - SRC
- File not found - -NAME
- File not found - META.IN
- File not found - -TYPE
- File not found - F
- File not found - -EXEC
- File not found - SH
- File not found - -C
- File not found - SED -E 'S/@VERSION@/1.9.8/G'         -E 'S/@REQUIRES@//G'    \$1\
- File not found -  > \${1%.IN}\
- File not found - SH
- File not found - {}
- File not found - ;
- Detecting compiler arguments: (extractor built) ok
```

What’s going on at the end of that snippet? It turns out ocamlfind’s `configure`
script contains a call to what it expects to be POSIX [`find`](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/find.html),
but what it’s actually getting is the Windows `find` utility. This problem is as
old as the hills, and affects `sort` as well:

```
C:\Users\DRA>find /?
Searches for a text string in a file or files.

FIND [/V] [/C] [/N] [/I] [/OFF[LINE]] "string" [[drive:][path]filename[ ...]]

  /V         Displays all lines NOT containing the specified string.
  /C         Displays only the count of lines containing the string.
  /N         Displays line numbers with the displayed lines.
  /I         Ignores the case of characters when searching for the string.
  /OFF[LINE] Do not skip files with offline attribute set.
  "string"   Specifies the text string to find.
  [drive:][path]filename
             Specifies a file or files to search.

If a path is not specified, FIND searches the text typed at the prompt
or piped from another command.
```

versus:

```
C:\Users\DRA>C:\msys64\usr\bin\find --version
find (GNU findutils) 4.10.0
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <https://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Written by Eric B. Decker, James Youngman, and Kevin Dalley.
Features enabled: D_TYPE O_NOFOLLOW(enabled) LEAF_OPTIMISATION FTS(FTS_CWDFD) CBO(level=1)
```

Now, it turns out that the OP’s system is misconfigured - which is part of the
reason why [Cygwin](https://cygwin.com), [MSYS2](https://www.msys2.org) and
[Git for Windows](https://gitforwindows.org/) don’t put their Unix shell
utilities into `PATH` by default. The Unix utilities are in `C:\msys64\usr\bin`,
but this entry in `PATH` appears after `C:\WINDOWS\system32`, so a few
utilities get shadowed. For things like `curl` and `tar`, this is _usually_
benign, but for `find` and `sort` it’s pretty terminal!

In its default mode, where opam manages the Cygwin or MSYS2 environment for
you - it’s part of a mildly complex process in [ocaml/opam#5832](https://github.com/ocaml/opam/pull/5832),
as opam ensures that either the Cygwin or MSYS2 `bin` directory is just “left”
enough in `PATH` to ensure that some key utilities are not shadowed, but not so
far to the left that it shadows things like Git for Windows!

However, if you’ve set-up `PATH` yourself, as the OP had, you’re on your own,
I'm afraid!

So why is all this making me rant about ShellCheck? Well, the problem is that
_downgrading_ ocamlfind to a previous version works just fine, and so it’s the
regression in ocamlfind 1.9.8 that I want to focus on. The failing part of the
script is:

```sh
# create META from META.in in POSIX-compatible & safe way
# see: https://www.shellcheck.net/wiki/SC2044
meta_subst="sed -e 's/@VERSION@/$version/g' \
        -e 's/@REQUIRES@/${req_bytes}/g' \
        \"\$1\" > \"\${1%.in}\""
find src -name 'META.in' -type f -exec sh -c "$meta_subst" sh {} \;
```

but which was originally (from [cbc4a7d](https://github.com/ocaml/ocamlfind/commit/cbc4a7db226670c12ee2891213593559bd694bbf)):

```sh
for part in `cd src; echo *`; do
    if [ -f "src/$part/META.in" ]; then
        sed -e "s/@VERSION@/$version/g" \
            -e "s/@REQUIRES@/${req_bytes}/g" \
            src/$part/META.in >src/$part/META
    fi
done
```

ShellCheck reports two issues with that original loop:
1. It dislikes the [use of backticks rather than `$(...)`](https://www.shellcheck.net/wiki/SC2006)
2. It worries that the [exit code of `cd` is unchecked](https://www.shellcheck.net/wiki/SC2164)

The first is because this script is very old, and it was written at a time when
there were not only versions of `sh` which didn’t support `$(...)` notation,
there were people using OCaml on them! In fact, it’s only comparatively recently
that the default version of `sh` in (Open)Solaris/Illumos, etc. actually
supports it.

The second is true, but:
1. It’s never going to happen (the `src` directory be missing from the
   tarball?!)
2. Even if it it does, the loop will _correctly_ do nothing, and the build will
   fail later

So, while ``for part in `cd src; echo *`; do`` is somewhat showing its age, in the
context of this specific fragment of this specific shell script, that will
**never ever** cause a problem. Writing a new script? Don't write that! Adding a
new part to an existing script? Don't write that!

However, it is now broken, and my proposed fix is that actually the use of the
wildcard was completely unnecessary, as we already compute the relevant list of
directories which can possibly contain a `META.in` file slightly later in the
script (taken from [1291259](https://github.com/ocaml/ocamlfind/pull/112/commits/129a259141fa3f1af157eef3100b74866bf90795)):

```sh
for part in $parts; do
  if [ -f src/"$part"/META.in ]; then
    sed -e "s/@VERSION@/$version/g" \
        -e "s/@REQUIRES@/${req_bytes}/g" \
        src/"$part"/META.in > src/"$part"/META
  fi
done
```

A piece of code got changed to fix neither a bug nor add a feature, and
something which worked before became broken. It took a non-trivial amount of
(wallclock) time to diagnose and fix the issue. That’s a lot of negative
consequences, and so far at least, there are _no_ positive consequences.

If it ain’t broke, don’t fix it!
