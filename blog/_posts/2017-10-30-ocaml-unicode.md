---
layout: post
title: "OCaml 4.06 Windows Unicode Support"
categories: platform
tags: "ocaml"
---
# Windows Unicode Support in OCaml 4.06.0

In August 2005, [Mantis Report #3771](https://caml.inria.fr/mantis/view.php?id=3771) was opened against OCaml 3.08.4 reporting that Windows applications could not read filenames containing Unicode characters. Various patches have been proposed over the last twelve years leading to [GitHub Pull Request #153](https://github.com/ocaml/ocaml/pull/153) almost exactly 2 years ago. This last spring, Clément Franchini at [Altair](http://www.altair.com) rebased the patch and added tests. Finally, Nicolás Ojeda Bär at [LexiFi](https://www.lexifi.com) took on the challenge of completing the patch and minimising the diff with a view to merging in OCaml 4.06.0.

The resulting [GitHub Pull Request #1200](https://github.com/ocaml/ocaml/pull/1200) just made it in time for the feature freeze last month, with a few follow-up pull requests since to polish things up. With luck (and testing!), we have a good working story with the 4.06.0 release candidate, which was pushed yesterday.

## Building OCaml from source

When I'm not hacking either [OCaml](https://github.com/ocaml/ocaml), [JBuilder](https://github.com/janestreet/jbuilder) or something else [OCamlLabs](http://ocamllabs.io)-related, I keep working towards a proper Windows implementation of opam 2.0. There are various solutions for opam already out there, but for trying out the new native Unicode, you may have an easier time being directly in the Windows Console for everything except the build and building entirely from sources.

The Unicode changes should work on any supported version of Windows, but for 4.06.0 you will only see UTF-8 sequences being correctly interpreted on Windows 10. On earlier versions, you can run `chcp 65001` to enable interpreting UTF-8 sequences, but you will quickly hit a bug in Windows which is tracked in [Mantis Report #6925](https://caml.inria.fr/mantis/view.php?id=6925). This means that prior to Windows 10, if you enable `CP_UTF8` with the `chcp 65001` command, you will see non-ASCII characters printing correctly, but following them you will also see some garbage characters from the UTF-8 sequences themselves. There is a patch for this in [GitHub Pull Request #1408](https://github.com/ocaml/ocaml/pull/1408) which will hopefully be in 4.07.0, due for release next year.

So, on your fresh Windows 10 machine, you'll need a build environment and for this we still recommend Cygwin. You can download its setup program from [cygwin.com](http://www.cygwin.com/setup-x86_64.exe) (or the [32-bit version](http://www.cygwin.com/setup-x86.exe), if you insist!). The only non-default package required is `make` - I'm going to build with Visual Studio 2017, but you can add the `mingw64-i686-gcc-core` and/or `mingw64-x86_64-gcc-core` packages if you want to build the mingw ports instead. The full command line for installing is:

```
setup-x86_64.exe --quiet-mode --no-shortcuts --no-startmenu --no-desktop --only-site --root C:\cygwin64 --site http://mirrors.kernel.org/sourceware/cygwin --packages make
```

or, if you prefer to type less (and want mingw):

```
setup-x86_64 -qnNdO -R C:\cygwin64 -s http://mirrors.kernel.org/sourceware/cygwin -P make,mingw64-x86_64-gcc-core
```

You'll also want to pick up [Visual Studio 2017](https://www.visualstudio.com/downloads/). For OCaml, we need the `VC++ 2017 v141 toolset (x86,x64)` component which gives the C compiler and linker, and then also the `Windows 10 SDK (10.0.16299.0) for Desktop C++ [x86 and x64]` component which gives us `Windows.h` and all the import libraries. It's not compulsory but I'd also select the `Visual Studio C++ core features` component, as this is where the shortcuts for starting the Developer Command Prompts are buried! Alternatively, you can install everything from the command line:

```
vc_community.exe --passive --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 --add Microsoft.VisualStudio.Component.Windows10SDK.16299.Desktop --add Microsoft.VisualStudio.Component.VC.CoreIde
```

Finally, let's download the [OCaml 4.06.0+rc1](http://caml.inria.fr/pub/distrib/ocaml-4.06/ocaml-4.06.0+rc1.tar.gz) tarball and also [FlexDLL 0.37](https://github.com/alainfrisch/flexdll/archive/0.37.tar.gz). FlexDLL has also had to be updated in order to support Unicode filenames, but this version also includes a fix for [Visual Studio 2017 Update 3](https://github.com/alainfrisch/flexdll/pull/40). Note that FlexDLL 0.37 is fully backwards compatible, and you can build versions of OCaml prior to 4.06.0 with it too.

OCaml needs to be built using Cygwin's make but with the Microsoft C Compiler available. The easiest way to set this up is to start an `x64 Native Tools Command Prompt for VS2017` and then run `C:\cygwin64\bin\mintty -` (note the `-` which starts a login shell). Let's leave the Command Prompt for a bit and use the bash session which opened and run the following commands:

```bash
$ cd /cygdrive/c/Users/DRA/Documents
$ tar -xzf ocaml-4.06.0+rc1.tar.gz
$ cd ocaml-4.06.0+rc1
$ eval $(tools/msvs-promote-path)
```

The configuration for OCaml has been changed slightly, so `s.h` and `m.h` must be copied to a new location:

```bash
$ cp config/m-nt.h byterun/caml/m.h
$ cp config/s-nt.h byterun/caml/s.h
```

The `Makefile` still resides in `config/Makefile`, so `cp config/Makefile.msvc64 config/Makefile`. At this stage, we need to edit `config/Makefile` and change `PREFIX` to the install location. I've been using `C:/Бактріан-406`, and I also configured a 4.05.0 build to `C:/Бактріан-405` so we can see some of the improvements side-by-side with the last release.

I always prefer to build flexlink as part of the main compilation, so we need to drop the sources in the right place:

```bash
$ tar -xzf ../flexdll-0.37.tar.gz
$ mv flexdll-0.37/* flexdll/
```

and finally we can now run:

```bash
$ make flexdll world.opt install
```

At this point, I'd take a coffee break, and once the build has finished and installed you can exit the bash shell, as we're now done with Cygwin.

You can even **build** OCaml in a directory which includes non-ASCII characters now, and we've started doing that on AppVeyor, as doing so revealed a few bugs (see the 🐫-infested logs at [AppVeyor](https://ci.appveyor.com/project/avsm/ocaml/build/1.0.4836)!).

## Looking at the improvements

Going back to the Command Prompt we started with, we can update its `PATH` to include our new compiler:

```
set Path=%PATH%;C:\Бактріан-406\bin
```

Now let's see the difference between OCaml 4.05.0 installed to `C:\Бактріан-406` and OCaml 4.06.0+rc1 installed to `C:\Бактріан-406`; just running `ocamlc -v`:

```
C:\Users\DRA\Documents\Unicode>ocamlc -v
The OCaml compiler, version 4.05.0
Standard library directory: C:/╨æ╨░╨║╤é╤Ç╤û╨░╨╜-405/lib
```

but in 4.06.0+rc1:

```
C:\Users\DRA\Documents\Unicode>ocamlc -v
The OCaml compiler, version 4.06.0+rc1
Standard library directory: C:/Бактріан-406/lib
```

You can already see the benefits of the console support on Windows 10. However, for 4.05.0, that's just a cosmetic problem, the real problem comes when we try to start the OCaml 4.05.0 toplevel:

```
C:\Users\DRA\Documents\Unicode>ocaml
C:\????????-405\bin\ocaml.exe not found or is not a bytecode executable file
```

Oops! So, at this point I'm going to switch to a different OCaml 4.05.0 compiler which I've instead installed to `C:\OCaml-405`. I've also setup a directory containing just one other directory called `Верблюд`.

Compare the effect of running `ocaml` and entering `Sys.readdir "."`

![Unicode Filenames]({{ site.url }}/assets/2017-10-30/2017-10-30-ocaml-unicode-1.png)

This even works through the environment:

```
C:\Users\DRA\Documents\Unicode\Ööç>set FOO=Верблюд

C:\Users\DRA\Documents\Unicode\Ööç>ocaml
        OCaml version 4.06.0+rc1

# Sys.chdir (Sys.getenv "FOO");;
- : unit = ()
# Sys.getcwd ();;
- : string = "C:\\Users\\DRA\\Documents\\Unicode\\Ööç\\Верблюд"
```

where 4.05.0 falls straight over:

```
C:\Users\DRA\Documents\Unicode\Ööç>set FOO=Верблюд

C:\Users\DRA\Documents\Unicode\Ööç>ocaml
        OCaml version 4.05.0

# Sys.chdir (Sys.getenv "FOO");;
Exception: Sys_error "???????: Invalid argument".
```

and even on the command line (run `echo print_string Sys.argv.(1) > foo.ml` to get `foo.ml`)

```
C:\Users\DRA\Documents\Unicode\Ööç>ocaml foo.ml Верблюд
Верблюд
```

Try that with 4.05.0 and you'll get a segfault!

## Technical details

What's actually happening under the hood is that we've switched all Windows system calls from the ancient Windows 9x-compatible ANSI calls to the modern Windows NT "Wide" Unicode versions (say hello to 1989!). This means that strings communicated to the operating system are represented as a sequence of 16-bit numbers instead of 8-bit numbers. Note that while sections of Windows have used UTF-16 since Windows 2000, it's better to consider the encoding as UCS-2, since this is what the Console displays - the Windows Console is not capable of displaying Unicode code points which would be encoded as surrogate pairs in UTF-16 (code points U+10000 and above).

However, this is not compatible with OCaml's string type and indeed is also not compatible with Unix-based OCaml. So although internally we're using UCS-2, what we expose to OCaml is UTF-8 which means that anything filename-related now expects to receive UTF-8 and will return UTF-8.

If your application previously dealt with ASCII filenames only, then everything will of course work, because UTF-8 is backwards compatible with ASCII. However, if you were using non-ASCII characters (with all the locale encoding problems which go with it), then the handling is different for filenames you provide to the runtime, but these are all extremely unusual filenames. For now, therefore, if a filename received by a runtime function is not valid UTF-8, then the runtime interprets the string as being 8-bit encoded in the current locale (which is effectively what the runtime did before). Note that runtime functions will still always return UTF-8.

If your application must behave in the old way, for example if you're concerned that you may have some filenames which do look like valid UTF-8 but should not be interpreted as such (for example, stored in a database or other data files) or if receiving UTF-8 encoded filenames back from the runtime is a problem, then it's possible to force the runtime to disable the UTF-8 handling by setting `WINDOWS_UNICODE=0` in `config/Makefile` (i.e. you need to build an entirely separately OCaml). In this mode, internally the runtime continues to use the "Wide" versions of the Windows system calls, but instead of translating strings to and from UTF-8, the runtime uses the current locale, which was the previous behaviour.

C bindings on Windows which handle filenames will not break per se, but you may end up passing a UTF-8 encoded string to a Windows ANSI function which doesn't recognise them. However, it is possible to access the functions the runtime is using to convert strings. In C, we've defined a new type `char_os` which is a normal `char` on Unix but a 16-bit `wchar_t` on Windows. There are functions provided for converting a UTF-8 string to and from a `char_os` string (`caml_stat_strdup_to_os` and `caml_stat_strdup_of_os`) and also an optimised function for converting a `char_os` string to an OCaml UTF-8 encoded string `caml_copy_string_of_os`. More details on this experimental API are available in the [manual](http://polychoron.fr/ocaml-beta-manual/4.06+rc1/intfc.html#sec473).
