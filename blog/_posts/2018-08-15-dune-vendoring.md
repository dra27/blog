---
layout: post
title: "The joys of Dune vendoring"
categories: platform
tags: "ocaml dune mirage"
draft: true
---
Of the many wonderful features provided by [Dune](https://github.com/ocaml/dune), my personal favourite remains its composability. It doesn't gain much mention in the manual because, well, there's not *that* much to explain about it beyond the simple fact that if you put separate Dune projects in subdirectories underneath your current project, then Dune will use those projects in preference to opam-installed libraries. This allows for a far superior workflow to developing using opam pins, as can be seen from my own recent addition of features to Mirage's [logs-syslog](https://github.com/hannesm/logs-syslog) library, which also required an addition to one of its dependencies, [syslog-message](https://github.com/verbosemode/syslog-message). Neither of these libraries are presently built using Dune, however porting [them](https://github.com/dra27/logs-syslog/tree/dune) [both](https://github.com/dra27/syslog-message/tree/dune) was not particularly difficult, and I thought it would be interesting to show how much more efficient the process of developing my recent patches would have been with Dune, rather than with `opam pin`.

# Background

syslog-message is a library which implements [RFC 3164](https://tools.ietf.org/html/rfc3164) and converts between an OCaml record for syslog messages and the (UDP) wire format. [Logs](http://erratique.ch/software/logs) provides a full infrastructure for logging which cleanly separates the generating of log events from their reporting. logs-syslog glues these two libraries together, making the syslog protocol a reporter for the Logs library, and providing the additional protocol work for sending syslog messages over both (unencrypted) TCP and TLS links.

logs-syslog was missing the ability to do local logging via `/dev/log` which I happened to want for a project. Sure, you can trivially configure rsyslog to listen on UDP, but it seemed better to add the option for Unix domain sockets.

I was working with a toy example in `log_mes.ml`:

{% highlight ocaml linenos %}
(* Install the syslog reporter *)
let () =
  match Logs_syslog_unix.tcp_reporter Unix.inet_addr_loopback () with
    | Error e -> prerr_endline e
    | Ok r -> Logs.set_reporter r
;;

Logs.set_level ~all:true (Some Logs.Debug);;

let src = Logs.Src.create "application:";;

Logs.warn ~src (fun m -> m "Hello syslog!");;
{% endhighlight %}

which with this `dune` file:

```
(executable
  (name log_mes)
  (libraries unix logs-syslog.unix))
```

can be built with `dune build log_mes.exe`

The change I wanted to make was to add a function `unix_reporter` which would take a `string` instead of an `inet_addr` and would replace the call in the `match`. So I cloned logs-syslog into my test directory, created a branch and added my `unix_reporter` function. I checked that the library built and then from the root of the clone I issued:

```sh
opam pin add logs-syslog . --kind=path
```

I used `--kind=path` as otherwise opam would pick up the fact it was a Git repository and create a git pin and I expected to have made some mistakes which I'd want to correct without having to commit each time, as that's even more typing. opam dutifully pinned logs-syslog and of course recompiled it in the process.

I then rebuilt my toy example using

```ocaml
  match Logs_syslog.unix_reporter () with
```

instead of the `tcp_reporter` before and tried to run it. At this point, I was getting entries in `/var/log/messages`, but not quite the ones I was after. After a bit of head-scratching and a lot of cursing at systemd's expense, I discovered that the wire protocol differs slightly when submitting syslog messages on Unix domain sockets. This meant I couldn't use the `Syslog_message.encode` function from syslog-message, but either needed to add another parameter or another function. I opted to add `Syslog_message.encode_local`. So I cloned syslog-message into my test directory, created a branch, added `encode_local` and also changed the call in logs-syslog to use that instead of `encode`. Then I issued:

```sh
opam pin add syslog-message . --kind-path
```

opam kindly offered to recompile logs-syslog at the same time. Then I rebuilt my test program, though it still didn't work properly. Ah, silly me! I'd forgotten to update the logs-syslog pin and opam quite correctly didn't refresh that when it was pinning syslog-message. So I issued:

```sh
opam upgrade logs-syslog
```

and waited while the pin was recompiled. Then I rebuilt my test program again and... wohoo! Success.

So, let's just recap those steps, noting the builds too:

 1. Cloned logs-syslog and added my new feature. **Build it to test that it's working**
 2. Pinned logs-syslog to my clone. **opam builds the library again**
 3. Build my test program and discovered that it's not quite right
 4. Cloned syslog-message and added my new feature. **Built it to test that it's working**
 5. Pinned syslog-message to my clone. **opam builds syslog-message and logs-syslog again**
 6. Build my test program and discovered it's still not working
 7. Realise that the pin for logs-syslog needed upgrading too. **opam builds logs-syslog again**
 8. Build my test program and finally have success

Obviously I could have omitted making mistakes, the point here is that a mistake could be made which then requires more builds. My test program also doesn't use either lwt, tls or mirage, but if my opam switch has those installed then those libraries will be being rebuilt each time by opam, even though I don't need them.

Now let's repeat the story using a version of those libraries which has been ported to Dune. Starting again with just `log_mes.ml` and that simple `dune` file (Dune will have generated a `dune-project` file too, but we don't need to worry about that). I now clone logs-syslog into my test directory and this time merge in my port to Dune in addition to adding my `unix_reporter` function (this is the old version before adding `encode_local` to syslog-message). From my test directory, I then simply run:

```
dune build --display=short log_mes.exe
```

I use `--display=short` here to show the commands being executed:

```
    ocamldep .log_mes.eobjs/log_mes.ml.d
    ocamldep logs-syslog/src/.logs_syslog.objs/logs_syslog.ml.d
    ocamldep logs-syslog/src/.logs_syslog.objs/logs_syslog.mli.d
    ocamldep logs-syslog/src/.logs_syslog_unix.objs/logs_syslog_unix.ml.d
    ocamldep logs-syslog/src/.logs_syslog_unix.objs/logs_syslog_unix.mli.d
      ocamlc logs-syslog/src/.logs_syslog.objs/logs_syslog.{cmi,cmti}
      ocamlc logs-syslog/src/.logs_syslog_unix.objs/logs_syslog_unix.{cmi,cmti}
      ocamlc .log_mes.eobjs/log_mes.{cmi,cmo,cmt}
    ocamlopt logs-syslog/src/.logs_syslog.objs/logs_syslog.{cmx,o}
    ocamlopt logs-syslog/src/logs_syslog.{a,cmxa}
    ocamlopt .log_mes.eobjs/log_mes.{cmx,o}
    ocamlopt logs-syslog/src/.logs_syslog_unix.objs/logs_syslog_unix.{cmx,o}
    ocamlopt logs-syslog/src/logs_syslog_unix.{a,cmxa}
    ocamlopt log_mes.exe
```

There are two really important things to take away from this: firstly, the building of the new logs-syslog library and my test application happened at the same time and, where possible, they take advantage of **inter-dependency** parallelism. You can see this at the start - `ocamldep` was being called for my test application *while logs-syslog was still being built*. The other thing to note is that while my opam switch includes all the depopts to build the mirage and lwt versions of logs-syslog, because my test program doesn't use them, they weren't compiled. In fact, in this build, there was even still a syntax error in the lwt version of logs-syslog! Finally, it's not wasted time building the bytecode version of logs-syslog either, because the test program is being compiled by ocamlopt.

Now I discover that my test program still isn't working properly, so I clone syslog-message and add `encode_local` and also merge in my port to Dune. At the same time, I update logs-syslog to use `encode_local`. Then from the test directory I run `dune build --display=short log_mes.exe` again:

```
    ocamldep syslog-message/src/.syslog_message.objs/syslog_message.mli.d
    ocamldep logs-syslog/src/.logs_syslog_unix.objs/logs_syslog_unix.ml.d
    ocamldep syslog-message/src/.syslog_message.objs/syslog_message.ml.d
      ocamlc syslog-message/src/.syslog_message.objs/syslog_message.{cmi,cmti}
      ocamlc logs-syslog/src/.logs_syslog.objs/logs_syslog.{cmi,cmti}
      ocamlc logs-syslog/src/.logs_syslog_unix.objs/logs_syslog_unix.{cmi,cmti}
      ocamlc .log_mes.eobjs/log_mes.{cmi,cmo,cmt}
    ocamlopt logs-syslog/src/.logs_syslog.objs/logs_syslog.{cmx,o}
    ocamlopt .log_mes.eobjs/log_mes.{cmx,o}
    ocamlopt syslog-message/src/.syslog_message.objs/syslog_message.{cmx,o}
    ocamlopt logs-syslog/src/logs_syslog.{a,cmxa}
    ocamlopt syslog-message/src/syslog_message.{a,cmxa}
    ocamlopt logs-syslog/src/.logs_syslog_unix.objs/logs_syslog_unix.{cmx,o}
    ocamlopt logs-syslog/src/logs_syslog_unix.{a,cmxa}
    ocamlopt log_mes.exe
```

Wow! As before, there's truly minimal rebuilding going on here. Unlike in opam's case, where it must rebuild the entire library, Dune holds all the cards - it knows, for example, that `logs_syslog.ml` and `logs_syslog.mli` were not changed, so it doesn't run `ocamldep` over them again. It's also building components of all three projects in parallel - both dependencies and the test program.

Now, it's time to upstream those changes. Because the libraries themselves are Dune projects, the build on them can be tested separately from where they are. For example, from within the `syslog-message` directory, running `dune build --display=short @install` gives:

```
Entering directory '/home/dra/syslog/test'
      ocamlc syslog-message/src/.syslog_message.objs/syslog_message.{cmo,cmt}
      ocamlc syslog-message/src/syslog_message.cma
    ocamlopt syslog-message/src/syslog_message.cmxs
```

Dune now completes the missing parts of syslog-message (the bytecode library and native code plugin), re-using all the native code from the earlier builds. You can do the same thing with logs-syslog to build the optional libraries. At this point, the new features get pushed to GitHub and pull requests can be made.

# Don't knock opam!

At this point, this is where pinning comes back: while waiting for releases, those packages can, and should, be pinned! Dune's awesomeness doesn't end here, though - once you're certain that everything's pushed, you can simply delete the clones from the test directory and add the pins:

```sh
# Please be sure you've pushed everything before doing this...
rm -rf syslog-message logs-syslog
opam pin add logs-syslog git+https://github.com/dra27/logs-syslog.git#unix-sockets --no-action
opam pin add syslog-message git+https://github.com/verbosemode/syslog-message.git#master --no-action
opam install syslog-message logs-syslog
```
 
Then run `dune build --display=short log_mes.exe` and:

```
      ocamlc .log_mes.eobjs/log_mes.{cmi,cmo,cmt}
    ocamlopt .log_mes.eobjs/log_mes.{cmx,o}
    ocamlopt log_mes.exe
```

Dune rebuilds just the binaries - it knows that the only thing which has changed is where the libraries are coming from, so it doesn't bother doing any `ocamldep` calls.

Remember, there are just two kinds of OCaml project: those which have switched to Dune, and those which need to!
