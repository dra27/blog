---
layout: post
title: "Dipping toes into OxCaml"
categories: platform
tags: "ocaml oxcaml"
---
[Jane Street](https://opensource.janestreet.com/) have been working for a few
years on a whole suite of extensions to OCaml, many of which they've both
blogged and published about. I did some hacking last year getting a version of
that running on Windows, which I really must resurrect. But today I actually had
a go at doing a tiny something with its features!

Stack allocation is a fascinating feature to add for me. I strongly believe that
OCaml's strength lies in pragmatism, and the promise of stack allocated values
is that we'll be able to write highly memory-performant code in OCaml _when we
want to_ (i.e. unlike in Rust, when we really don't care, we can just leave it
all to the GC as normal) and without having to compromise massively that code.

I've dusted off the first day of [Advent of Code 2024](https://adventofcode.com/2024/day/1).
Initially, not looking at solving the actual puzzle, but my input is 1000 lines
of text where each line is two 5 digit numbers separated by three spaces. Here's
a trivial snippet over that:

```ocaml
let f a _s = succ a

let execute file =
  In_channel.with_open_text file (In_channel.fold_lines f 0)

let () =
  let c1 = Gc.minor_words () in
  let r = execute "input-01" in
  let c2 = Gc.minor_words () in
  Printf.printf
    "Result: %d\n%.0f words allocated across the call\n" r (c2 -. c1)
```

This is just counting the number of lines and for me is showing 3012 words
allocated on the minor heap. There are 1000 lines in the file each of which
needs 14 bytes (including the terminator) so, until one of our GC experts
corrects me, I reckon that's 1000 headers, 2000 words containing the strings
themselves and we can wave our hands about the channel and closure in those
calls to account for the other 12 words.

So far, so good - this is just counting the lines. Now, as a further toy example
(if one were really concerned about performance, this is totally not the way to
do this...), let's add them up instead:

```ocaml
let f a s =
  let fst = String.sub s 0 5 in
  let snd = String.sub s 8 5 in
  a + int_of_string fst + int_of_string snd
```

That gives me 7012 minor words - another 4000, which corresponds to all of those
`String.sub` calls (2000 6-byte strings, 2000 header words). So what can stack
allocation bring us? Well, I'm wanting to "lift the bonnet" with all this, so
rather than using Base, let's have a little bit of hand-rolled support (I said
this was a toy example):

```ocaml
module String = struct
  include String

  external unsafe_create_local : int -> local_ bytes = "caml_create_local_bytes"

  external unsafe_blit_string :
    (string[@local_opt]) -> int -> (bytes[@local_opt]) -> int -> int -> unit
      = "caml_blit_string" [@@noalloc]

  external unsafe_to_string :
    (bytes[@local_opt]) -> (string[@local_opt]) = "%bytes_to_string"

  external get :
    (string[@local_opt]) -> (int[@local_opt]) -> char = "%string_safe_get"

  let sub_local s ofs len = exclave_
    if ofs < 0 || len < 0 || ofs > length s - len
    then invalid_arg "String.sub"
    else begin
      let r = unsafe_create_local len in
      unsafe_blit_string s ofs r 0 len;
      unsafe_to_string r
end
```

What's interesting to me is that this doesn't look _too_ different from an
expanded version of `String.sub` from the Standard Library:

```ocaml
let sub s ofs len =
  if ofs < 0 || len < 0 || ofs > length s - len
  then invalid_arg "String.sub / Bytes.sub"
  else begin
    let r = create len in
    unsafe_blit s ofs r 0 len;
    unsafe_to_string r
  end
```

we just had to _choose_ to create the stack-allocated strings (yes, yes, stack
allocated is still allocated, which isn't necessary, of course). But we can now
plug that in:

```ocaml
let f a s =
  let fst = String.sub_local s 0 5 in
  let snd = String.sub_local s 8 5 in
  a + int_of_string fst + int_of_string snd
```

and:

```
   |   a + int_of_string fst + int_of_string snd
                         ^^^
Error: This value escapes its region.
```

Ah, interesting to see how it spreads: we need an updated `int_of_string`:

```ocaml
external int_of_string : (string[@local_opt]) -> int = "caml_int_of_string"
```

and now it works _and we're back to the same allocations as when counting the
lines instead_!

All the mode inference works as you'd expect too: rewriting it so that `f` takes
the `sub` function as an argument:

```ocaml
let f sub a s =
  let fst = sub s 0 5 in
  let snd = sub s 8 5 in
  a + int_of_string fst + int_of_string snd

let execute sub file =
  In_channel.with_open_text file (In_channel.fold_lines (f sub) 0)

let show name sub =
  let c1 = Gc.minor_words () in
  let r = execute sub "input-01" in
  let c2 = Gc.minor_words () in
  Printf.printf
    "Result for %s: %d\n%.0f words allocated across the call\n"
    name r (c2 -. c1)

let () =
  show "sub" String.sub;
  show "sub_local" String.sub_local
```

and still 4000 words fewer on the minor heap for `String.sub_local`. Tiny first
impression: the `[@local_opt]` annotation feels quite infectious for library
authors!

More serious playing to come... who knows, might even re-do the puzzle!
