---
layout: post
title: "Adventures with BuildKit"
categories: platform
tags: "ocaml windows"
---

I've been doing battle the last few days with Docker, and in particular trying
to persuade BuildKit to do what I wanted. I find Docker leans towards being a
deployment tool, rather than a development tool which is to say that it's
exceedingly useful for both, but when I encounter problems trying to persuade it
to do what I'm after for development, it tends to feel I'm not using it for the
purpose for which it was intended.

Anyway, maybe documenting the journey will reveal how much of this view is my
own ignorance and it will definitely consolidate a few useful tricks in one
place ready for next time.

Docker shines when I'm at the stage of needing to test multiple configurations
or versions of what I'm doing against one bit of code that I'm working on. Its
[multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
provide a very convenient and tidy way to fan out a single build tree into
multiple configurations (versus, say, using multiple worktrees, etc.) and the
[BuildKit](https://docs.docker.com/build/buildkit/) backend adds parallelism.
Couple of that with an unnecessarily large number of CPU cores, more RAM than
existed in the world when I was a child, and many terrabytes of cache, and
you're sorted!

I've been working on meta-programming the installation targets for OCaml's build
system to allow them to do things other than simply installing OCaml (generating
opam `.install` files, cloning scripts and so forth). The commit series for that
got plugged into the branch set for Relocatable OCaml and fairly painlessly
backported. It's all GNU make macros and so forth - no type system helping and
various bits that have shifted around over the past few releases. I'd devised a
series of manual tests for the branch [against trunk OCaml](https://github.com/dra27/ocaml/commits/opam-install-file),
a little bit of glue to generate a `Dockerfile`, and the testing against the
backports could be automated. Our [base images](https://images.ci.ocaml.org) are
a useful starting point:

```Dockerfile
FROM ocaml/opam:ubuntu-24.04-opam AS base

RUN sudo apt-get update && sudo apt-get install -y gawk autoconf2.69
RUN sudo apt-get install -y vim

ENV OPAMYES="1" OCAMLCONFIRMLEVEL="unsafe-yes" OPAMPRECISETRACKING="1"
RUN sudo ln -f /usr/bin/opam-2.3 /usr/bin/opam && opam update

RUN git clone https://github.com/dra27/ocaml.git
WORKDIR ocaml
```

That sets up an image we can then use as a fanout for running the actual tests,
which is then a whole series of (generated) fragments. The first bit sets up the
compiler before my changes:

```Dockerfile
FROM base AS test-4.14-relocatable
RUN git checkout 32d46126b2b993a7ac526a339c85d528d3a280cd || git fetch origin && git checkout 32d46126b2b993a7ac526a339c85d528d3a280cd
RUN ./configure -C --prefix $PWD/_opam --docdir $PWD/_opam/doc/ocaml --enable-native-toplevel --with-relative-libdir=../lib/ocaml --enable-runtime-search=always --enable-runtime-search-target
RUN make -j
RUN make install
RUN mv _opam _opam.ref
```

The `git checkout foo || git fetch origin && git checkout foo` is a neat little
bit of Docker fu: first try to checkout the commit you need and only if that
fails do a Git pull. That means that if something gets changed while developing,
only the containers which need to pull will do so, preserving caching (if we
re-did the clone in `base`, it'd invalidate _all_ the builds so far).

Then it actually does the battery of tests:

```Dockerfile
RUN git checkout e1794e2548a1e8f6dc11841b0ac9ad159ca89988 || git fetch origin && git checkout e1794e2548a1e8f6dc11841b0ac9ad159ca89988
RUN make install && diff -Nrq _opam _opam.ref && rm -rf _opam
RUN git checkout 86ecf4399873045d7eca03560d9ac84eebae38e8 || git fetch origin && git checkout 86ecf4399873045d7eca03560d9ac84eebae38e8
RUN if grep ...
RUN if test -n ...
RUN git checkout 671122db576cb0e6531cf1fa3b18af225f840c36 || git fetch origin && git checkout 671122db576cb0e6531cf1fa3b18af225f840c36
RUN if grep '^ROOTDIR *=' * -rIl ...
RUN git checkout fbf12456dd47d758d1858bd6edf8dd3310a7ca3b || git fetch origin && git checkout fbf12456dd47d758d1858bd6edf8dd3310a7ca3b
RUN if grep 'INSTALL_\(DATA\|PROG\)' ...
RUN make install && diff -Nrq _opam _opam.ref && rm -rf _opam
RUN if test -n "$(make INSTALL_MODE=list ...
RUN make INSTALL_MODE=display install
RUN make INSTALL_MODE=opam OPAM_PACKAGE_NAME=ocaml-variants install
RUN make INSTALL_MODE=clone OPAM_PACKAGE_NAME=ocaml-variants install
RUN test ! -d _opam
RUN opam switch create . --empty && opam pin add --no-action --kind=path ocaml-variants .
RUN opam install ocaml-variants --assume-built
```

The nifty part is that if one individual branch needed tweaking, the script to
generate the `Dockerfile` puts the new commit shas in there and BuildKit then
rebuilds just the parts needed. The whole thing then just needs tying together
with something that forces the builds to be "necessary":

```Dockerfile
FROM base AS collect
WORKDIR /home/opam
COPY --from=test-4.08-vanilla /home/opam/ocaml/config.cache cache-4.08-vanilla
COPY --from=test-4.08-relocatable /home/opam/ocaml/config.cache cache-4.08-relocatable
COPY --from=test-4.09-vanilla /home/opam/ocaml/config.cache cache-4.09-vanilla
COPY --from=test-4.09-relocatable /home/opam/ocaml/config.cache cache-4.09-relocatable
COPY --from=test-4.10-vanilla /home/opam/ocaml/config.cache cache-4.10-vanilla
COPY --from=test-4.10-relocatable /home/opam/ocaml/config.cache cache-4.10-relocatable
...
COPY --from=test-5.2-relocatable /home/opam/ocaml/config.cache cache-5.2-relocatable
COPY --from=test-5.3-vanilla /home/opam/ocaml/config.cache cache-5.3-vanilla
COPY --from=test-5.3-relocatable /home/opam/ocaml/config.cache cache-5.3-relocatable
COPY --from=test-5.4-vanilla /home/opam/ocaml/config.cache cache-5.4-vanilla
COPY --from=test-5.4-relocatable /home/opam/ocaml/config.cache cache-5.4-relocatable
COPY --from=test-trunk-vanilla /home/opam/ocaml/config.cache cache-trunk-vanilla
COPY --from=test-trunk-relocatable /home/opam/ocaml/config.cache cache-trunk-relocatable
```

The purpose of that last step is just to extract _something_ from all the other
containers to force them to be built. [It worked really nicely](https://github.com/dra27/relocatable/tree/main/test-install),
the testing identified a few slips here and there with the commit series, and it
was very efficient to re-test it after any tweaks.

So... having got that working, I wanted to make sure that changes I'd made to
the [monster script](https://github.com/dra27/relocatable/commits/main/stack)
that reconstitutes Relocatable OCaml back at the beginning of the month were
working on all of the older lock files. Partly because things should be _always_
be reproducible, but also because I have needed to go back to older iterations
of Relocatable OCaml, I added a lockfile system to it last year. For example,
[ef758648dd](https://github.com/dra27/ocaml/commit/ef758648dd743bd471d17d8183a3ee5b6d9da61b)
describes the exact branches which contributed to the OCaml Workshop 2022 talk
on Relocatable OCaml. It takes a list of branch commands:

```
fix-autogen@4.08 6b37fcefa88a21f5972ca64e1af89e060df6a83c
fcommon@4.08 2c36ba5c19967b69c879bc0a9f5336886eb8df6b
sigaltstack 044768019090c2aeeb02b4d0fb4ddf13d75be8c6
sigaltstack-4.09@fixup 8302a9cd4f931f232e40078048d02d35a7075f05
fix-4.09.1-configure@4.09 7e1f5a33e0cdd3f051a5c5ab76f1d097270e232e
install-bytecode@4.08 1287da77f952166e1c60d93da0e756b2ba7d33b7
win-reconfigure@4.08 162af3f1ff477a6a0e34816fe855ef474c07b273
mingw-headers@4.09 78e3c94924b07ff2941a6313b35fca8bd0fc7ce1
makefile-tweaks@4.09 6a2af5c14176e06275ff4da7dc6a14fd4f49093a
static-libgcc@4.11 260ec0f27682822f255f8cf64cf4e4faa6fa8088
config.guess@4.11 7efc39d9bcb943375c35dd024c60e21c8fecda6a
config.guess-4.09@4.09 185183104b4d559eb5f24fc1d0d2531976f1ee0e
fix-binutils-2.36@4.11 5b1560952044faee8b2502b3595c0598e7402513
fix-mingw-lld@4.11 5443fec22245ff37fda7e2ce8ad554daf11fa0df
...
```
and crunches that to produce a commit series for each required version:
```
backport-5.0 b5c11faed67511e25a2ee9cac953362b6b165a37
backport-4.14 0598df18732107619f4d500f9c372e648b6c0174
backport-4.13 f2cd54453f7c4684af8fdb2c2c1d4b14119d077f
backport-4.12 de72889271d8875589a0e9690ab220f9ffcc4eb1
backport-4.11 a15f4a165ae27929fec94e05b65257126883eafd
backport-4.10 e95093194d0ec378de3d86033bd011b3d8cb7eb2
backport-4.09 4ee19334d40a5a5c0a69de53a8e77eb3f6fc5829
backport-4.08 52ec6c2f54e9d8c0fb950e7b4a2016ec9a624756
```
Now, it should always be possible to build a given lock with the `stack` script
from the date it was made, but it's actually more useful to be able to build it
with the latest one - the problem is that occasionally things go wrong. So... I
have a `Dockerfile` a la the one above which tests whether each lock is still
buildable.

So, what I'd hoped was going to work was to put each lock in an image, just like
with the testing, build it with a "known good" version of the script, then add
additional `RUN` lines to each of the images to use a newer version of the
`stack` script and then debug as I went, being able to take advantage of the
bootstrap caching from the previous stages so that it wouldn't be tortuously
slow. Docker seemed to have other ideas, though. I guess because there were so
many artefacts flying around, some of those intermediate layers were being
evicted from the cache. I tried cranking up `builder.gc.defaultKeepStorage`, but
to no avail. I switched to the containerd imagge storage backend and tried using
`--cache-to`, which allows cranking the cache aggressiveness with `mode=max`.
That seemed to work, but at the cost of waiting ages at the end of each build
for all the intermediate to be exported.

I'd just about given up, but then I had an idea to turn the problem on its HEAD:
instead of fighting Docker and trying to convince it that all these intermediate
builds were precious, how about making it that the final container (the
"collect" bit) actually contained all the artefacts? In this case, the most
"precious" artefact that's wanted is any bootstraps of OCaml done as part of the
commit series - they're computationally expensive to perform, and the `stack`
script already has a trick where it scours the `reflog` looking for previous
instances of the same bootstrap. The `base` stage is similar to the previous
test - but before fanning out, this time another `builder` stage is added:

```Dockerfile
FROM base AS builder
RUN <<End-of-Script
  git clone --shared relocatable build
  cd build
  git submodule init ocaml
  git clone /home/opam/relocatable/ocaml --shared --no-checkout .git/modules/ocaml
  mv .git/modules/ocaml/.git/* .git/modules/ocaml/
  rmdir .git/modules/ocaml/.git
  cp ../relocatable/.git/modules/ocaml/hooks/pre-commit .git/modules/ocaml/hooks/
  git submodule update ocaml
  cd ocaml
  git remote set-url origin https://github.com/dra27/ocaml.git
  git remote add --fetch upstream https://github.com/ocaml/ocaml.git

  ...
End-of-Script
WORKDIR /home/opam/build/ocaml
```

There's some fun Git trickery combining with Docker caching. The `base` stage
did the main clone - so `/home/opam/relocatable` is a normal clone of
[dra27/relocatable](https://github.com/dra27/relocatable) and then
`/home/opam/relocatable/ocaml` is an initialised submodule cloning
[dra27/ocaml](https://github.com/dra27/ocaml) and also with [ocaml/ocaml](https://github.com/ocaml/ocaml)
fetched. That's a lot of stuff, and `/home/opam/relocatable/.git/modules/ocaml`
is 562M. So the `builder` stage does two tricks: firstly it clones the _local_
copy of relocatable again, but using `--shared`. Then it does a similar trick
with the submodule (for some reason I couldn't get to the bottom, while
`git submodule update` supports most of `git clone`'s obscure arguments, it
doesn't support `--shared`, so the trick with moving things around does the
clone for it. The result of that is a copy of the relocatable clone, but with
none of the commits copied. That's subtly different from using worktrees - it
means that each parallel build will exactly store just the new commits it adds
into its git repo. That means 50-350MB per image, instead of 600-950MB, so a
considerable saving.

The trick then is to copy those Git clones back as part of the collection stage:

```Dockerfile
FROM base AS collector
COPY --chown=opam:opam --from=lock-818afcc496 /home/opam/build/.git/modules/ocaml builds/818afcc496/.git
COPY --from=lock-818afcc496 /home/opam/build/log logs/log-818afcc496
RUN sed -i -e '/worktree/d' builds/818afcc496/.git/config
COPY --chown=opam:opam --from=lock-727272c2ee /home/opam/build/.git/modules/ocaml builds/727272c2ee/.git
COPY --from=lock-727272c2ee /home/opam/build/log logs/log-727272c2ee
RUN sed -i -e '/worktree/d' builds/727272c2ee/.git/config
COPY --chown=opam:opam --from=lock-8d9989f22a /home/opam/build/.git/modules/ocaml builds/8d9989f22a/.git
COPY --from=lock-8d9989f22a /home/opam/build/log logs/log-8d9989f22a
RUN sed -i -e '/worktree/d' builds/8d9989f22a/.git/config
COPY --chown=opam:opam --from=lock-032059697e /home/opam/build/.git/modules/ocaml builds/032059697e/.git
COPY --from=lock-032059697e /home/opam/build/log logs/log-032059697e
...
```

Of course, that quickly resulted in too many layers, so in fact it's fanned out
into a series of "collector" images so that at the end of the build, the
directory `builds` contains the Git repository from each of the builds, but not
any source artefacts. That can then all be plumbed into the _original_ repo to
create the final image:

```Dockerfile
FROM base
COPY --chown=opam:opam --from=collector /home/opam/builds builds
COPY --chown=opam:opam --from=collector /home/opam/logs logs
COPY --from=reflog /home/opam/HEAD .
RUN cat HEAD >> relocatable/.git/modules/ocaml/logs/HEAD && rm -f HEAD
COPY <<EOF relocatable/.git/modules/ocaml/objects/info/alternates
/home/opam/builds/ef758648dd/.git/objects
/home/opam/builds/b026116679/.git/objects
/home/opam/builds/511e988096/.git/objects
...
/home/opam/builds/590e211336/.git/objects
/home/opam/builds/b5aa73d89c/.git/objects
EOF
WORKDIR /home/opam/relocatable/ocaml
RUN <<End-of-Script
  cat >> rebuild <<"EOF"
  head="$(git -C ../../builds/ef758648dd rev-parse --short relocatable-cache)"
  for lock in b026116679 511e988096 d2939babd4 be8c62d74b c007288549 ...; do
    while IFS= read -r line; do
      args=($line)
      if [[ ${#args[@]} -gt 2 ]]; then
        parents=("${args[@]:3}")
        head=$(git show --no-patch --format=%B ${args[0]} | git commit-tree -p $head ${parents[@]/#/-p } ${args[1]})
      fi
    done < <(git -C ../../builds/$lock log --format='%h %t %p' --first-parent --reverse relocatable-cache)
  done
  git branch relocatable-cache $head
EOF
  bash rebuild
  rm rebuild
  for lock in ef758648dd b026116679 511e988096 d2939babd4 ...; do
    script --return --append --command "../stack $lock" ../log
  done
End-of-Script
```

Et voilà! One final image that contains all those precious bootstraps unified
and where the storage overhead of the parallel builds is kept to a minimum...
and, as a result of that, BuildKit's cache seems to be working for me, rather
than against 🥳
