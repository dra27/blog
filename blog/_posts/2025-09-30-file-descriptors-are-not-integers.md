---
layout: post
title: "File descriptors are not integers"
categories: platform
tags: "ocaml windows"
---

There was a flurry of activity on [ocaml-multicore/ocaml-uring](https://github.com/ocaml-multicore/ocaml-uring)
this month leading to a release ([ocaml/opam-repository#28604](https://github.com/ocaml/opam-repository/pull/28604)).
ocaml-uring provides bindings to the Linux's [io\_uring](https://github.com/axboe/liburing),
which allows batching various syscalls to the kernel for it to execute
out-of-order, and in parallel. Its principal use at the moment is for the
high-performance Linux backend of [Eio](https://github.com/ocaml-multicore/eio).

Various of the syscalls available in io\_uring return Unix file descriptors, and
the design of ocaml-uring as a low-level interface to it leads to some slightly
unfortunate recommendations in its instructions:

```ocaml
# let fd =
    if result < 0 then failwith ("Error: " ^ string_of_int result);
    (Obj.magic result : Unix.file_descr);;
val fd : Unix.file_descr = <abstr>
```

Anil was wondering if we could dust off some of the code in an old failed OCaml
pull request of mine ([ocaml/ocaml#1990](https://github.com/ocaml/ocaml/pull/1990))
to get rid of some of the magic. Pulling out the C code from that PR wasn't
entirely mechanical, and Anil wondered if we might put it in a library. As it
happened, there'd been some discussions about the PR at developer meetings, and
there was a consensus that we ought to have an official C API for converting a
C file descriptor (i.e. an `int`) to an OCaml `Unix.file_descr`. I wasn't that
keen on producing an external library without sorting out the compiler's library
at the same time, so I figured I'd dust that change off and see where it went.

That original PR attempted to add primitives to the Unix module to allow OCaml
code to convert an _OCaml_ `int` to `Unix.file_descr`, so the ocaml-uring
example would instead just be `Unix.descr_of_fd result` with no `Obj.magic`.
However, that approach had an absolute veto from [Xavier](https://github.com/ocaml/ocaml/pull/1990#issuecomment-413464113):

_“File descriptors are not integers”_

I started off on this little rabbit-hole of changes agreeing philosophically,
but not really agreeing, but ended up in total agreement, and ending with the
feeling that the fact they are integers - and that OCaml treats integers
specially - encourages possibly poorer library design.

On Unix, a `Unix.file_descr` is just the OCaml representation of the C `int`,
but it's absolutely not that on Windows, where the implementation is much more
complicated. So the `Obj.magic` "trick" is a route to a segfault on Windows.
That's obviously not important for a Linux-only library like ocaml-uring, but
`Obj.magic` is a wart. The Windows complexity exists because we have both CRT
file descriptors (which are just C `int`s, the same as on Unix) and also OS
file handles (which are Win32 `HANDLE`s - a pointer). There's some added
book-keeping complexity needed, but that's not important right now. The key
thing is that we have a notion of an "Operating System" file descriptor and a
"C Runtime Library" file descriptor. On Unix, they happen to be the same thing;
on Windows, they're not. On Windows, given one, it is always possible to obtain
the other (the functions [`_get_osfhandle`](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/get-osfhandle)
and [`_open_osfhandle`](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/open-osfhandle)
are provided for this.

That gives a straightforward portable C API: `caml_unix_file_descr_of_os` which
takes an `int` (on Unix) or a `HANDLE` (on Windows) and returns an OCaml
`Unix.file_descr` representing it. It's similarly straightforward to provide
`caml_unix_file_descr_of_fd` and `caml_unix_fd_of_file_descr` for getting the
CRT file descriptor on both, and we then have the portable primitives required
(`caml_unix_os_of_file_descr` is not necessary, but the reasons are left as an
exercise for the avid portable C stub code author).

To this, I then added a quick commit to solve another issue in this area, which
is tracked in [ocaml/ocaml#9052](https://github.com/ocaml/ocaml/issues/9052).
While there's the veto on _converting_ a `Unix.file_descr` to an `int`, being
able to debug the values in logs and so forth is useful.

At this point, the scab of this old PR well and truly picked, I paused and
thought a bit about the original problem I'd been trying to solve in [#1990](https://github.com/ocaml/ocaml/pull/1990),
and which remained not-entirely-satisfactorily solved. It's also described in
[ocaml/ocaml#6948](https://github.com/ocaml/ocaml/issues/6948), and while
looking through it, I realised [some of my previous work]({% post_url 2025-04-03-cloexec %})
was related to this. In particular, that passing specific file descriptors from
one process to another is _not_ a Unix-specific operation, and you can do it on
Windows as well, it's just less common (i.e. you can start a Windows process
with file descriptor 3 connected to a control channel if you want, just as you
can on Unix).

Which got me thinking some more about having an OCaml function for this, and
about “file descriptors are not integers”, and I realised that while this is
_mostly_ true, it's not _always_ true. For a start, there are three well-known
values, 0 for "input", 1 for "output", and 2 for "logging", known as the
"Standard Input, Output and Error" handles. These are exposed in the Unix
library as `Unix.stdin`, `Unix.stdout` and `Unix.stderr` (and in the Standard
Library itself, for the channels API), and you can also specify them when
spawning processes. Stepping back, let's consider if instead of treating file
descriptors as `int`, [Thompson](http://cs.bell-labs.co/who/ken/) and [Ritchie](https://9p.io/who/dmr/index.html)
had instead used [opaque pointers](https://en.wikipedia.org/wiki/Opaque_pointer#C)[^note]
for `open`, `read`, and so forth in the first version of Unix.

In this hypothetical scenario, virtually all C programs would be just fine:
`STDIN_FILENO` and so forth would still be there, and virtually all C code using
file descriptors doesn't actually care about the precise value. However, the
implementation would face the same issue as we have in OCaml when trying to
spawn other processes if we needed to _pass_ file descriptors to a process. At
that point, it dawned on me: “file descriptors are **indeed** not integers**,
but at program startup, there is a partial map from integers to file
descriptors**”. Most processes are created with at least `0`, `1` and `2` added
to that map, and they can then be retrieved with `Unix.stdin`/`STDIN_FILENO`,
etc. Given that Windows _does_ support the notion of passing additional file
descriptors, that means that this operation **is portable**, and the inability
to pass _abstract_ file descriptors **using** _integers_ seems like a gap. This
fits with a TODO item I noted [in April]({% post_url 2025-04-03-cloexec %}):
once `Unix.create_process` et al are correctly _inheriting_ file descriptors on
Windows, it makes even more sense to be able to do this too.

[#6948](https://github.com/ocaml/ocaml/issues/6948) proposed adding a
`file_descr list` argument to functions like `create_process`, but that's the
wrong API here. It _assumes_ that each descriptor is being passed using its
current file descriptor number, which is wrong for two reasons. Firstly, it
breaks the abstraction (we have just treated a `Unix.file_descr` as an `int`).
Secondly, as noted in the C API above, if the `Unix.file_descr` was created from
an OS file descriptor on Windows, it doesn't necessarily _have_ a CRT descriptor
associated with it. The fix is easy: pass a `(int * file_descr) list`. We
preserve the abstraction, and specify the mapping.

With this extra list, when we spawn a process, we now have the ability to pass,
say, the input portion of a pipe (created with `Unix.pipe`) as file
descriptor 3. How can we pick that up on the OCaml side? As it happens, on
Windows we really could trivially enumerate them, and so have
`Unix.startup_file_descrs : (int * file_descr) list` or some such, but there's
no way to do that portably on Unix. For this specific case, there is a concrete
reason to have a function `int -> Unix.file_descr` (indeed, this is exactly what
the Unix library does internally to create `stdin`, `stdout` and `stderr`).
However, there is _no case_ for a function `Unix.file_descr -> int`.

What to call this function, therefore? Having `file_descr_of_fd` (as in C) but
not `fd_of_file_descr` seems very strange. But what does the type tell us? We're
taking an `int` - a _low-level_ descriptor and returning a `file_descr` - a
_high-level_ descriptor. C already has two levels - the stream API ("`FILE *`")
and provides a function to go from the low level to the stream API. It's
[`fdopen`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/fdopen.html).
The C streams API isn't really about abstraction, so C also has a function to go
the other way, [`fileno`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/fileno.html),
but the nice property is that `fdopen` and `fileno` do not sound like reverse
operations.

Fixing `Unix.create_process` on Windows is definitely a job for another day, but
adding `Unix.fdopen` is easy, so I did that. The three commits are sat in
[dra27/ocaml#fdopen](https://github.com/dra27/ocaml/commits/fdopen). Next up,
applying all this to ocaml-uring to get rid of the magic…

[^note]: I couldn't find a precise reference for exactly when in the early 1970s this became possible, relative to the first releases of Unix!
