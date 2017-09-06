---
layout: post
title: "Windows Console Performance (or lack thereof)"
categories: platform
tags: "ocaml windows opam"
---
I spent a few days at the weekend refreshing my native Windows fork of Opam and while doing so, found an interesting performance issue for console applications on Windows.

One of the jobs porting both Opam and opam-repository for native Windows is updating all the OCaml compiler packages themselves to have Windows build commands. It's quite extensive, since the instructions have varied across versions and there are also a lot of [patches](https://github.com/metastack/ocaml-legacy) which need to be applied, so this is done using an `opam-admin.top` script. This little helper program in the Opam developer tools includes a useful function `Opam_admin_top.iter_packages` which passes every package definition in a repository along with its `opam` file to a function you provide and writes any changes to that opam back to the repository.

There are ~7750 packages processed and the utility displays the name of each on a single status line (writing `\r` and then a terminal line erase sequence, the latter has to be emulated on older versions of Windows with lots of spaces...). Windows text mode applications use a subsystem called the Console Host which is generally slower than your average Unix terminal. The repeated text `Processing package` flickered lots and in a brief moment of silliness, I decided to investigate further (I was running the command a lot, while tweaking the script), though the results were surprising. On the Windows 7 VM I still use for Opam development, the script was taking ~50 seconds to run, but if the status output was redirected to a file, it shaved a massive 20 seconds off the execution time!

So opam-admin.top now has some rate limiting code for the status messages - the script now only takes half a minute to run, and the output even looks nicer. Moral of the story is clear: never write more to the console than you have to.

This needs a little further investigation - it'll be interesting to see if the Windows 10 console host offers any improvements and it would be good to have comparison numbers for a Linux distro.
