---
layout: post
title: "File descriptors as integers considered harmful"
categories: platform
tags: "ocaml windows"
---
Continuing [the previous foray into file descriptors]({% post_url 2025-09-30-file-descriptors-are-not-integers %}),
and trying to remove `Obj.magic` both from ocaml-uring's code itself, and from
the recommended way for using it.

I expected this to be a fairly simple application of the C API features added in
[dra27/ocaml#file\_descr-interop](https://github.com/dra27/ocaml/commits/file_descr-interop).
It's not quite what happened, though - as I don't actually think that any of the
changes I propose for the Unix library in OCaml are needed for ocaml-uring. What
piqued my interest in this rather deep rabbit-hole was that what I would contend
are slightly poor design decisions stem from the corrupting notion in C that
file descriptors are integers - i.e. because file descriptors in C _are_
integers, it becomes dangerously tempting to keep them as that in OCaml, because
OCaml [treats integers specially](https://ocaml.org/manual/5.3/intfc.html#s%3Ac-value).

[ocaml-uring's documentation](https://ocaml.org/p/uring/latest/doc/index.html)
states that it "aims to provide a thin type-safe layer for use in higher-level
interfaces". I think adopting Xavier's _veto_ on manipulating file descriptors
as integers _in OCaml_ (i.e. _never_ doing it) leads to greater type safety,
even in thin bindings, without compromising performance. Let's see! 🤓

The [sources for ocaml-uring](https://github.com/ocaml-multicore/ocaml-uring)
use `Obj.magic` both in the library itself in one place, and at various points
in both the tests and the documentation, exclusively for duping the type system
into believing an `int` is a `Unix.file_descr`.

Consider [mkdirat(2)](https://man7.org/linux/man-pages/man2/mkdirat.2.html).
This is part of a family of \*at syscalls which accept a file descriptor as an
argument and which will use the directory name of the file referred to by that
file descriptor as the base for a relative path. There is a special value
`AT_FDCWD` which can be passed instead of an open file descriptor (it instructs
`mkdirat` instead to interpret paths relative to the current working directory),
and it's the representation of this `AT_FDCWD` which causes the `Obj.magic`.
When used in C, `AT_FDCWD` is an illegal file descriptor value (it's -100, as it
happens). Continuing with `mkdirat` as an example, we have this C declaration:

```c
void io_uring_prep_mkdirat(struct io_uring_sqe *sqe,
                           int dirfd, const char *path, mode_t mode);
```

and [this](https://github.com/ocaml-multicore/ocaml-uring/blob/main/lib/uring/uring.ml#L341)
corresponding primitive in ocaml-uring:
```ocaml
external submit_mkdirat : t -> id
                          -> Unix.file_descr -> Sketch.ptr -> int
                          -> bool = "ocaml_uring_submit_mkdirat" [@@noalloc]
```

There's a close correspondence between the two - the OCaml `t` gets passed to
[`io_uring_get_sqe`](https://man7.org/linux/man-pages/man3/io_uring_get_sqe.3.html)
which gives us the first parameter for the prep call. The `id` is separately
passed to [`io_uring_set_sqe_data`](https://man7.org/linux/man-pages/man3/io_uring_sqe_set_data.3.html)
(that's part of a mechanism in liburing to be able to identify which syscalls
have completed - more on that later). The remaining arguments in this primitive
(the `Unix.file_descr`, the `Sketch.ptr` and the `int`) correspond to the
`dirfd`, `path` and `mode` arguments to the syscall (the liveness of buffers is
complicated here - the `Sketch.ptr` type is essentially allowing a string to be
passed from OCaml to C without subsequently having to worry about it being moved
by the GC).

It looks like a good thin layer over the syscall. Every one of these prep calls
will necessarily have a `io_uring_set_sqe_data` call, and it's always good to
minimise the number of crossings between C and OCaml when writing C bindings.

It naturally becomes tempting to have a `Unix.file_descr` value for `AT_FDCWD`,
and that's indeed [the implementation](https://github.com/ocaml-multicore/ocaml-uring/blob/main/lib/uring/uring.ml#L468):

```ocaml
let at_fdcwd : Unix.file_descr = Obj.magic Config.at_fdcwd
```

However, there really is no need for this, and if we step back from the fact
it's an `int`, and instead consider what `AT_FDCWD` is _modelling_, a more
obvious solution emerges which doesn't require magic. `AT_FDCWD` is essentially
be used to mean "no file descriptor" (and there's then an interpretation for
what that means `mkdirat` should do). i.e. `AT_FDCWD` means `None`, which is
then being modelled by -100. An actual file descriptor is `Some fd`, which is
being modelled by its number (which can never be -100). In other words, we could
pass a `Unix.file_descr option` to the C stub, instead of a `Unix.file_descr`.
This falls out even more naturally when we look at the actual [`Uring.mkdirat`](https://github.com/ocaml-multicore/ocaml-uring/blob/main/lib/uring/uring.ml#L498-L502)
exposed to the library user:

```ocaml
let mkdirat t ~mode ?(fd=at_fdcwd) path user_data =
```

It actually already has a `Unix.file_descr option` because it's using an
optional argument to convey it! It's quite an [easy adjustment](https://github.com/dra27/ocaml-uring/commit/00ca9ecd88216ae4fdd039d92c58619686e4b4af)
just to allow the C stub to resolve this, it has the benefit that the value of
`AT_FDCWD` no longer has to be determined from OCaml, and it doesn't involve any
additional allocations of memory on the OCaml side (because the optional
argument meant there was already one).

So far, so hopefully good. Now on to the one in the README:

```ocaml
# let fd =
    if result < 0 then failwith ("Error: " ^ string_of_int result);
    (Obj.magic result : Unix.file_descr);;
val fd : Unix.file_descr = <abstr>
```

Well, it wasn't my intention, but the proposed `Unix.fdopen` [could deal with
that](https://github.com/dra27/ocaml-uring/commit/6c14ffd68be586e48fa4293df76338db32a28c12).
Except that this is not "`fdopen`" in the spirit it was added for - in fact,
it's precisely what was _not_ supposed to be done. We've got an `int` in OCaml
that is _actually_ a file descriptor, and we're trying to coerce it to be a
`Unix.file_descr`. We've got a layering violation - we've allowed an abstract
value from the world of C that is represented in C as an `int` to leak up to the
world of OCaml as an OCaml `int`, which should be a number!

Funnily enough, ocaml-uring has another minor layering violation like this in
the translation of Unix error values to `Unix.error` values in the
[`Uring.error_of_errno` function](https://github.com/ocaml-multicore/ocaml-uring/blob/main/lib/uring/uring.ml#L651-L652):

```ocaml
let error_of_errno e =
  Uring.error_of_errno (abs e)
```

I didn't want to abuse my proposed `Unix.fdopen` this way, so soon after
conceiving it for a more noble and even slightly-typed purpose, so instead I
opted for bouncing these numbers back to C by instead adding [`Uring.file_descr_of_result`](https://github.com/dra27/ocaml-uring/pull/3/files):

```c
value ocaml_uring_file_descr_of_result(value v_result)
{
  CAMLparam0();
  CAMLlocal2(result, val);
  if (Int_val(v_result) < 0) {
    val = unix_error_of_code(-Int_val(v_result));
    result = caml_alloc_small(1, 1);
  } else {
    val = caml_unix_file_descr_of_fd(Int_val(v_result));
    result = caml_alloc_small(1, 0);
  }
  Field(result, 0) = val;
  CAMLreturn(result);
}
```

which handles the pattern in C, where it belongs. The result certainly looks
nicer in some of the test code, for example, the numeric matching in one of the
tests:

```ocaml
if result >= 0 then
  let fd = Unix.fdopen result in
  Unix.close fd
else
  let error = Uring.error_of_errno fd in
  raise (Unix.Unix_error(error, (* ... *)))
```

becomes a slightly more idiomatic:

```ocaml
match Uring.file_descr_of_result result with
| Ok fd -> Unix.close fd
| Error error -> raise (Unix.Unix_error(error, (* ... *)))
```

but there are two things that still feel bad - those `int`s are _still there_
in OCaml! Worse, we're now incurring an extra C call to process them.

The veto tells us that the processing belongs in C. Is there a way to do this
where we don't _ever_ have the general `int` result exposed in OCaml, but we're
still a thin wrapper around the API _and_ at no cost?

Not without breaking the API, but I think we can have our API cake and also eat
it in this case. A first stab sits on [my fork](https://github.com/dra27/ocaml-uring/pull/2/files).
The result of an io\_uring operation is always a (C) `int`. Each of the syscalls
uses one of three encodings of that integer. In all three cases, a negative
number corresponds to an error, where the absolute value of the number is the
Unix error number (from `errno.h`). Some functions then just return 0 on
success. The other functions return a number ≥ 0 and for some of those, that
number is a file descriptor.

Now, the choice of encoding is known _when we prep the call_. So the trick I
tried instead was to encode that in the user-data attached to the ring entry.
The ocaml-uring implementation is using an array index as the user-data, so even
on a 32-bit system we can comfortably steal two bits to encode how to treat the
result. The three wait functions now instead of returning:

```ocaml
type cqe_option = private
| Cqe_none
| Cqe_some of { user_data_id : id; res: int }
```

return a slightly richer:

```ocaml
type cqe_option = private
| Cqe_none
| Cqe_unit of { user_data_id : id; result: unit }
| Cqe_int of { user_data_id : id; result: int }
| Cqe_fd of { user_data_id : id; result: Unix.file_descr }
| Cqe_error of { user_data_id : id; result: Unix.error }
```

A little bit of additional plumbing is needed for cancellation. I opted to have
the C functions themselves return the amended user-data value, as it's only
needed for cancellation. That keeps the considerations of C in C-land, but it
might be better to push that into the `Heap` implementation which backs the IDs
and allow both the job ID and heap pointer to be the same number.

I didn't push this all the way back to Eio, but I did update all the tests and
documentation, and I have to say the results looked nice to me (and still
low-level) - in particular, the copying tests seem to get a clearer delineation
of error path. I'm not sure if the separation of `unit`/`int` was strictly
buying much, and given that various of the error codes which may want to come
aren't Posix and so map to `Unix.UNKNOWNERR` (which allocates), it would
possibly be worth looking at a richer error type which could itself be converted
to a `Unix.error` expensively, but on-demand. However, what was nice was that
the return from the wait from functions still allocates just one two-field OCaml
word, and the file descriptor - if there is one - incurs no further conversion,
the value comes from C with the correct types already.

Entertainingly, the final version of the patch, while breaking the API, does not
actually need any alterations to OCaml's Unix library at all.

There's one last possibly fun OxCaml follow-up to look at. The value which comes
back from the C wait functions is a `cqe_option` (see above) whose shape is a
single constant constructor and four 2-field constructors. The first field in
these cases is the io\_uring user-data, which we then translate back to the
OCaml value to yield one of these:

```ocaml
type 'a completion_option =
  | None
  | Unit of { result: unit; data: 'a }
  | Int of { result: int; data: 'a }
  | FD of { result: Unix.file_descr; data: 'a }
  | Error of { result: Unix.error; data: 'a }
```

Which has the same shape! Now, we could do an evil `Obj.magic` trick here to
re-use the block and simply replace the field which contained the io\_uring user
data with the OCaml value (we'd need to swap the fields around in the types, but
that's a minor detail). However, this kind of lying can be dangerous when the
OCaml optimiser comes into play... but can the uniqueness modes be used here to
allow _the compiler_ to determine that the `cqe_option` block is now available
and actually compile it down to just that field assignment? We'd then halve the
allocations arising from a ring wait operation 🤔

Anyway, file descriptors are not integers. I'm a new disciple. I think we should
make T-Shirts.
