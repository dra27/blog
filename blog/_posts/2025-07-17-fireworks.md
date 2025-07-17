---
layout: post
title: "Fireworks and things"
categories: platform
tags: "ocaml windows"
---

Thanks to some targetted optimisations in the [script which manages Relocatable
OCaml's various branches](https://github.com/dra27/relocatable/commits/main/stack),
I'd vastly improved the turn-around time when making changes to the patch-set
and propagating them through the various tests and backports. On Tuesday night,
the _entire_ set of branches was green in CI (they're sat [here](https://github.com/dra27/ocaml/pulls?q=is%3Apr+is%3Aopen+label%3Arelocatable+combined)
with green check marks and everything). All that was to be needed on Wednesday
was to quickly update the opam packaging to take advantage of
Relocatable-awesomeness and plumb it all together. The 2022 version of the
packages for Ljubljana I knew contained a hack for searching a previous switch,
but I'd already investigated a more principled approach using opam's `build-id`
variable, so it would just be a matter of plumbing that in and using the cloning
mechanism already in that script.

And then I opened that scripts which I'd hacked together ready for the talk in
2022 in Ljubljana. I vaguely remember getting that all working at some ungodly
hour of the morning. The final clone is an unsightly:

```sh
    # Cloning
    SOURCE="$(cat clone-from)"
    cp "$SOURCE/share/ocaml/config.cache" .
    mkdir -p "$1/man/man1"
    cp "$SOURCE/man/man1/"ocaml* "$1/man/man1/"
    mkdir -p "$1/bin"
    cp "$SOURCE/bin/"ocaml* "$1/bin/"
    rm -rf "$1/lib/ocaml"
    cp -a "$SOURCE/lib/ocaml" "$1/lib/"
```

with no attempt to check the file lists 😭 Sorting out the installation targets
in OCaml's build system is on my radar, but was not on my "Relocatable OCaml
blockers" TODO list.

<div class="tenor-gif-embed" data-postid="22401840" data-share-method="host" data-aspect-ratio="1.74863" data-width="100%"><a href="https://tenor.com/view/were-so-close-gif-22401840">We were so close</a></div> <script type="text/javascript" async src="https://tenor.com/embed.js"></script>

Alas, the things which you can get away for a demo in a conference talk aren't
quite the same as for actually maintained software. Ho hum - on the plus side,
Wednesday and Thursday's hacking now yields a version of OCaml which can
generate opam install files properly, and which can therefore be co-opted to
produce the cloning script actually required. Onwards and upwards, apparently
now with memes...
