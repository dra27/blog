---
layout: post
title: "They do it with mirrors, you know - that sort of thing"
categories: platform
tags: "ocaml windows"
---
While comfort-watching the indomitable [Joan Hickson](https://en.wikipedia.org/wiki/Joan_Hickson)
as Agatha Christie's [Miss Marple](https://en.wikipedia.org/wiki/Miss_Marple_(TV_series))
in [The Body in the Library](https://en.wikipedia.org/wiki/The_Body_in_the_Library_(film)),
it occurred to me that Miss Marple would have been a formidable debugger. Since
returning from holiday ~~one~~, ~~two~~, three weeks ago, I've been mostly
straightening out and finalising the final Relocatable OCaml PR. A frustrating
task, because I know these things will take weeks and have little to show for at
the end, so one spends the entire time feeling it should be finished by now.
It's just about there, when this little testsuite failure popped up:

```
List of failed tests:
    tests/lib-unix/common/cloexec.ml
    tests/warnings/mnemonics.mll
```

In both cases there was a similar, very strange-looking error:

```
the file '/home/runner/work/ocaml/ocaml/testsuite/tests/lib-unix/_ocamltest/tests/lib-unix/common/cloexec/ocamlc.byte/cloexec_leap.exe' is not a bytecode executable file
```

and

```
the file '/home/runner/work/ocaml/ocaml/testsuite/tests/warnings/_ocamltest/tests/warnings/mnemonics/ocamlc.byte/mnemonics.byte' is not a bytecode executable file
Fatal error: exception File "mnemonics.mll", line 55, characters 2-8: Assertion failed
```

Now, as it happens, the diagnosis of _what_ was happening was relatively quick
for me. I've dusted off and thrown around so many obscure bits of the runtime
system on so many diverse configurations and platforms with Relocatable OCaml
that it's resulted in a lot of other bugs being fixed _before_ the main PRs,
some bugs fixed _with_ the main PRs and then a pile of follow-up work with the
additional parts. There's one particularly long-standing bug on Windows:

```
C:\Users\DRA>where ocamlc.byte
C:\Users\DRA\AppData\Local\opam\default\bin\ocamlc.byte.exe

C:\Users\DRA>where ocamlc.byte.exe
C:\Users\DRA\AppData\Local\opam\default\bin\ocamlc.byte.exe

C:\Users\DRA>ocamlc.byte.exe --version
5.2.0

C:\Users\DRA>ocamlc.byte --version
unknown option --version
```

Strange, huh: `ocamlc.byte.exe` does one thing and `ocamlc.byte` does another!
The precise diagnosis of what's going on there is nearly a novel in itself. The
fix is quite involved, and is at the “might get put into PR 3; might be left for
the future” stage. The failures across CI were just the Unix builds which use
the stub launcher for bytecode (it's an obscure corner of startup which lives in
[`stdlib/header.c`](https://github.com/ocaml/ocaml/tree/trunk/stdlib/header.c)
and which has received a pre-Relocatable overhaul in [ocaml/ocaml#13988](https://github.com/ocaml/ocaml/pull/13988)).
There are so many bits to Relocatable OCaml that I have a master script that
puts them all together and then backports them: the CI failure was only on the
"trunk" version of this, the 5.4, 5.3 and 5.2 versions passing as normal. The
backports don't include the "future" work, so that quickly pointed me at the
work sitting in [dra27/ocaml#190](https://github.com/dra27/ocaml/pull/190/commits).

Both those failures are from tests which themselves spawn executables as part of
the test. What was particularly strange was mnemonics because that doesn't call
itself, rather it calls the compiler:

```ocaml
let mnemonics =
  let stdout = "warn-help.out" in
  let n =
    Sys.command
      Filename.(quote_command ~stdout
                  ocamlrun [concat ocamlsrcdir "ocamlc"; "-warn-help"])
  in
  assert (n = 0);
```

That's invoking the `ocamlc` bytecode binary from the root of the build tree
passing it as an argument directly to `runtime/ocamlrun` in the root of the
build tree. The fact that ocamlrun is then displaying a message referring to
`mnemonics.byte` is very strange, but was down to a bug in my fix for this other
issue. The core of the bug-fix is that the stub launcher, having opened the
bytecode image to find its `RNTM` section so it can search for the runtime to
call now leaves the file descriptor open and hands its number over to `ocamlrun`
as part of the `exec` call (works on Windows as well). The problem was the
cleanup from this in `ocamlrun` itself, where that environment is reset having
been consumed:

```c
#if defined(_WIN32)
  _wputenv(L"__OCAML_EXEC_FD=");
#elif defined(HAS_SETENV_UNSETENV)
  unsetenv("__OCAML_EXEC_FD=");
#endif
```

There's a stray `=` at the end of the Unix branch there 🫣 Right, problem solved
and, were I Inspector Slack, I should have zipped straight round to Basil
Blake's gaudy cottage, handcuffs at the ready.

But what about the second murder? Which, in this case, is why the heck hadn't
this been seen before? That's the kind of thing that terrifies me with a fix
like this: the bug is obvious, but was something else being masked and, more to
the point, have I just changed something which introduced a _different_ bug
which happened to cause this one to be visible. At this point, I made a note,
closed my laptop, and returned to my knitting (no, wait, that was Miss Marple).
Then the penny dropped: the compiler's being configured here with
`--with-target-sh=exe` (on Unix, that means that bytecode executables
intentionally avoid shebang-style scripts and use the stub), which should mean
that those two tests are compiled using the stub. Except that because we test
the compiler in the build tree, previously the compiler picks up
`stdlib/runtime-launch-info` which is the _build_ version of that header, not
the _target_ version. However, one of the refactorings I've done in [c60e4aaf](https://github.com/dra27/ocaml/pull/189/commits/c60e4aafcf97bde037445e4cd94a9e659caf072a)
stops using `runtime-launch-info` this way (I introduced that header in [ocaml/ocaml#12751](https://github.com/ocaml/ocaml/pull/12751)
as part of OCaml 5.2.0). A side-effect of that change is that
`stdlib/runtime-launch-info` is actually the target version of the header, and
the _root_ bytecode compiler is _now_ behaving as we'd always been expecting it
to that test, using target configuration defined in `utils/config.ml`... and so
only now revealing this latent bug in my fix.

_“They do it with mirrors, you know-that sort of thing-if you understand me.”
Inspector Curry did not understand. He stared and wondered if Miss Marple was
quite right in the head._
