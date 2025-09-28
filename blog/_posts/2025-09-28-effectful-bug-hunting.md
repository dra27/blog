---
layout: post
title: "A second foray into agentic coding"
categories: platform
tags: "ocaml llm agentic"
---

Continuing the [previous theme](({% post_url 2025-09-17-late-to-the-party %}))
of dabbling with matters agentic. Previously, I'd quite assiduously kept my
fingers away from files. This time, I wanted to try something exploratory,
switching to the agent for things I was actively stuck on.

I was still (very) curious at the latent remaining bug in [Lucas's excellent work](({% post_url 2025-09-25-building-with-effects %})).
There were some corners which had been cut in the prototype, and I had a brief
foray into this problem, with a view this time to ensuring artefact equivalence
between what OCaml's build system would produce and what our altered driver
program was doing.

If you have a pre-built compiler and a clean (of binary artefacts) OCaml source
tree, you can actually build the bytecode compiler in just three, ahem, short
commands (I'm intentionally glossing over all the generated source files):
```console
$ ocamlc -I utils -I parsing -I typing -I bytecomp -I file_formats -I lambda -I middle_end -I middle_end/closure -I middle_end/flambda -I middle_end/flambda/base_types -I driver -I runtime -g -strict-sequence -principal -absname -w +a-4-9-40-41-42-44-45-48 -warn-error +a -bin-annot -strict-formats -linkall -a -o compilerlibs/ocamlcommon.cma utils/config.mli utils/build_path_prefix_map.mli utils/format_doc.mli utils/misc.mli utils/identifiable.mli utils/numbers.mli utils/arg_helper.mli utils/local_store.mli utils/load_path.mli utils/profile.mli utils/clflags.mli utils/terminfo.mli utils/ccomp.mli utils/warnings.mli utils/consistbl.mli utils/linkdeps.mli utils/strongly_connected_components.mli utils/targetint.mli utils/int_replace_polymorphic_compare.mli utils/domainstate.mli utils/binutils.mli utils/lazy_backtrack.mli utils/diffing.mli utils/diffing_with_keys.mli utils/compression.mli parsing/location.mli parsing/unit_info.mli parsing/asttypes.mli parsing/longident.mli parsing/parsetree.mli parsing/docstrings.mli parsing/syntaxerr.mli parsing/ast_helper.mli parsing/ast_iterator.mli parsing/builtin_attributes.mli parsing/camlinternalMenhirLib.mli parsing/parser.mli parsing/pprintast.mli parsing/parse.mli parsing/printast.mli parsing/ast_mapper.mli parsing/attr_helper.mli parsing/ast_invariants.mli parsing/depend.mli typing/annot.mli typing/value_rec_types.mli typing/ident.mli typing/path.mli typing/type_immediacy.mli typing/outcometree.mli typing/primitive.mli typing/shape.mli typing/types.mli typing/data_types.mli typing/rawprinttyp.mli typing/gprinttyp.mli typing/btype.mli typing/oprint.mli typing/subst.mli typing/predef.mli typing/datarepr.mli file_formats/cmi_format.mli typing/persistent_env.mli typing/env.mli typing/errortrace.mli typing/typedtree.mli typing/signature_group.mli typing/printtyped.mli typing/ctype.mli typing/out_type.mli typing/printtyp.mli typing/errortrace_report.mli typing/includeclass.mli typing/mtype.mli typing/envaux.mli typing/includecore.mli typing/tast_iterator.mli typing/tast_mapper.mli typing/stypes.mli typing/shape_reduce.mli file_formats/cmt_format.mli typing/cmt2annot.mli typing/untypeast.mli typing/includemod.mli typing/includemod_errorprinter.mli typing/typetexp.mli typing/printpat.mli typing/patterns.mli typing/parmatch.mli typing/typedecl_properties.mli typing/typedecl_variance.mli typing/typedecl_unboxed.mli typing/typedecl_immediacy.mli typing/typedecl_separability.mli lambda/debuginfo.mli lambda/lambda.mli typing/typeopt.mli typing/typedecl.mli typing/value_rec_check.mli typing/typecore.mli typing/typeclass.mli typing/typemod.mli lambda/printlambda.mli lambda/switch.mli lambda/matching.mli lambda/value_rec_compiler.mli lambda/translobj.mli lambda/translattribute.mli lambda/translprim.mli lambda/translcore.mli lambda/translclass.mli lambda/translmod.mli lambda/tmc.mli lambda/simplif.mli lambda/runtimedef.mli file_formats/cmo_format.mli middle_end/internal_variable_names.mli middle_end/linkage_name.mli middle_end/compilation_unit.mli middle_end/variable.mli middle_end/flambda/base_types/closure_element.mli middle_end/flambda/base_types/var_within_closure.mli middle_end/flambda/base_types/tag.mli middle_end/symbol.mli middle_end/flambda/base_types/set_of_closures_id.mli middle_end/flambda/base_types/set_of_closures_origin.mli middle_end/flambda/parameter.mli middle_end/flambda/base_types/static_exception.mli middle_end/flambda/base_types/mutable_variable.mli middle_end/flambda/base_types/closure_id.mli middle_end/flambda/projection.mli middle_end/flambda/base_types/closure_origin.mli middle_end/clambda_primitives.mli middle_end/flambda/allocated_const.mli middle_end/flambda/flambda.mli middle_end/flambda/freshening.mli middle_end/flambda/base_types/export_id.mli middle_end/flambda/simple_value_approx.mli middle_end/flambda/export_info.mli middle_end/backend_var.mli middle_end/clambda.mli file_formats/cmx_format.mli file_formats/cmxs_format.mli bytecomp/instruct.mli bytecomp/meta.mli bytecomp/opcodes.mli bytecomp/bytesections.mli bytecomp/dll.mli bytecomp/symtable.mli driver/pparse.mli driver/compenv.mli driver/main_args.mli driver/compmisc.mli driver/makedepend.mli driver/compile_common.mli utils/config.ml utils/build_path_prefix_map.ml utils/format_doc.ml utils/misc.ml utils/identifiable.ml utils/numbers.ml utils/arg_helper.ml utils/local_store.ml utils/load_path.ml utils/clflags.ml utils/profile.ml utils/terminfo.ml utils/ccomp.ml utils/warnings.ml utils/consistbl.ml utils/linkdeps.ml utils/strongly_connected_components.ml utils/targetint.ml utils/int_replace_polymorphic_compare.ml utils/domainstate.ml utils/binutils.ml utils/lazy_backtrack.ml utils/diffing.ml utils/diffing_with_keys.ml utils/compression.ml parsing/location.ml parsing/unit_info.ml parsing/asttypes.ml parsing/longident.ml parsing/docstrings.ml parsing/syntaxerr.ml parsing/ast_helper.ml parsing/ast_iterator.ml parsing/builtin_attributes.ml parsing/camlinternalMenhirLib.ml parsing/parser.ml parsing/lexer.mli parsing/lexer.ml parsing/pprintast.ml parsing/parse.ml parsing/printast.ml parsing/ast_mapper.ml parsing/attr_helper.ml parsing/ast_invariants.ml parsing/depend.ml typing/ident.ml typing/path.ml typing/primitive.ml typing/type_immediacy.ml typing/shape.ml typing/types.ml typing/data_types.ml typing/rawprinttyp.ml typing/gprinttyp.ml typing/btype.ml typing/oprint.ml typing/subst.ml typing/predef.ml typing/datarepr.ml file_formats/cmi_format.ml typing/persistent_env.ml typing/env.ml typing/errortrace.ml typing/typedtree.ml typing/signature_group.ml typing/printtyped.ml typing/ctype.ml typing/out_type.ml typing/printtyp.ml typing/errortrace_report.ml typing/includeclass.ml typing/mtype.ml typing/envaux.ml typing/includecore.ml typing/tast_iterator.ml typing/tast_mapper.ml typing/stypes.ml typing/shape_reduce.ml file_formats/cmt_format.ml typing/cmt2annot.ml typing/untypeast.ml typing/includemod.ml typing/includemod_errorprinter.ml typing/typetexp.ml typing/printpat.ml typing/patterns.ml typing/parmatch.ml typing/typedecl_properties.ml typing/typedecl_variance.ml typing/typedecl_unboxed.ml typing/typedecl_immediacy.ml typing/typedecl_separability.ml typing/typeopt.ml typing/typedecl.ml typing/value_rec_check.ml typing/typecore.ml typing/typeclass.ml typing/typemod.ml lambda/debuginfo.ml lambda/lambda.ml lambda/printlambda.ml lambda/switch.ml lambda/matching.ml lambda/value_rec_compiler.ml lambda/translobj.ml lambda/translattribute.ml lambda/translprim.ml lambda/translcore.ml lambda/translclass.ml lambda/translmod.ml lambda/tmc.ml lambda/simplif.ml lambda/runtimedef.ml bytecomp/meta.ml bytecomp/opcodes.ml bytecomp/bytesections.ml bytecomp/dll.ml bytecomp/symtable.ml driver/pparse.ml driver/compenv.ml driver/main_args.ml driver/compmisc.ml driver/makedepend.ml driver/compile_common.ml
$ ocamlc -I utils -I parsing -I typing -I bytecomp -I file_formats -I lambda -I middle_end -I middle_end/closure -I middle_end/flambda -I middle_end/flambda/base_types -I driver -I runtime -g -strict-sequence -principal -absname -w +a-4-9-40-41-42-44-45-48 -warn-error +a -bin-annot -strict-formats -a -o compilerlibs/ocamlbytecomp.cma bytecomp/bytegen.mli bytecomp/printinstr.mli bytecomp/emitcode.mli bytecomp/bytelink.mli bytecomp/bytelibrarian.mli bytecomp/bytepackager.mli driver/errors.mli driver/compile.mli driver/maindriver.mli bytecomp/instruct.ml bytecomp/bytegen.ml bytecomp/printinstr.ml bytecomp/emitcode.ml bytecomp/bytelink.ml bytecomp/bytelibrarian.ml bytecomp/bytepackager.ml driver/errors.ml driver/compile.ml driver/maindriver.ml
$ ocamlc -I utils -I parsing -I typing -I bytecomp -I file_formats -I lambda -I middle_end -I middle_end/closure -I middle_end/flambda -I middle_end/flambda/base_types -I driver -I runtime -g -compat-32 -o ocamlc -strict-sequence -principal -absname -w +a-4-9-40-41-42-44-45-48 -warn-error +a -bin-annot -strict-formats compilerlibs/ocamlcommon.cma compilerlibs/ocamlbytecomp.cma driver/main.mli driver/main.ml
```

I wanted to try a different angle on the `Load_path`, and this time produced a
function which _predicts_ the files in the tree. The rules for this were pretty
easy for me to define, and I wasn't sure I could face watching Claude
special-case everything. 130 lines of verifiably correct hacked OCaml later, I
had my load path function. A little bit more code later, those three commands
above were translated into an OCaml script (based on the ocamlcommon and
ocamlbytecomp libraries) which should exactly the same build. It ran - and it
built the compiler.

`ocamlc` was, pleasingly, _exactly_ the same. The .cma files, however, were not.
For ocamlcommon.cma, that turned out to be me being sloppy with my commands.
`ocamlcommon.cma` is linked with `-linkall`, but
`ocamlc -a foo.cma -linkall bar.cmo` is not the same as
`ocamlc -a foo.cma -linkall bar.ml`, because `-linkall` gets recorded in the
.cmo file _as well_. Easy fix - but the files were still different. A bit more
tweaking and I could see that actually the .cmo files were different.

A bit more poking and checking with `ocamlobjinfo` and a few other flags and
tricks, and I observed that:

```console
$ ocamlc -g -c utils/config.ml
```

resulted in slightly different _debug information_ from:

```console
$ console -g -c utils/config.mli utils/config.ml
```

(it's observably to do with the debug information - omit the `-g` and they're
all identical). Lots to suspect here, but time for...

```console
$ claude
╭───────────────────────────────────────────────────╮
│ ✻ Welcome to Claude Code!                         │
```

The problem was easy to state, but not quite so quick to come up with a
conclusive explanation. Claude, like most of these models, appears not to have
been trained on [this old cartoon](https://www.researchgate.net/figure/Then-a-Miracle-Occurs-Copyrighted-artwork-by-Sydney-Harris-Inc-All-materials-used-with_fig2_302632920),
and very merrily buzzes along for a few rounds of investigation, followed by a
highly dubious explanation for how it was probably something to do with
marshalling and, mumble mumble, the final binaries are the same so this bug is
probably OK.

Hmm. A few rounds of, "no, this needs to be equivalent as otherwise it's not
reproducible" ("You're so right!"), and we had a lot of test programs, a
frequent need for reminders that debugging OCaml's Marshalling format was
possibly not going to help, but we weren't very much closer to an answer.

Stepping back, I re-framed the problem, instead asking Claude to produce a
program which would give a textual dump of the debug information in each file,
so we could compare it. This was interesting - especially the occasional
hallucinations at having analysed "all the fields", but we got there.

What was interesting was that we were struggling to perceive differences between
anything. Claude at this point was desperate to delve into the runtime code and
start doing hex-dumps of the marshal format to see what was actually different.
I appear to be a little older than Claude, and was more reticent about this
approach. I suggested we look at the polymorphic hash of some of these fields
instead. At this point, we started to see some differences - Claude's inferences
at this point were working well, and there was a strong suggestion to add all
sorts of accessor functions into the `Types` module to be able to introspect
some of the values in more detail than normally intended (i.e. polymorphic hash
was telling that us that some abstract values were different, but we wanted to
see what the differences really were).

Reader, I told it to use `Obj.magic` instead 🫣

However, what happened next was truly fascinating and definitely very efficient.
The value being returned for one of the type IDs was simply not believable. It
was far too _high_. Claude also correctly observed that it was in fact a block,
and not an integer, which was what we were expecting. The human brain at this
point cuts in, and looks at the type: `Types.get_id: t -> int`. No, that
accessor looks right. Brain slowly whirring; look at the code:

```ocaml
let get_id t = (repr t).id
```

Oh - it's not an accessor (in another life, I could possibly have performed
Claude's responses...).

All I had to point out was that `Types.get_id` was not an accessor, it was
normalising the result (to walk `Tlink` members of the type representation), and
Claude was on it, replacing semi-elegant OCaml code with a sea of calls to `Obj`
functions.

But we had our answer - the type chain was different, if semantically equivalent
and, more importantly, Claude then leaped to the problem.

The internal `Types.new_id` reference isn't reset between compilations 💥

A quick rebuild later, and the same debug information was given regardless of
whether `utils/config.mli` was compiled at the same time as `utils/config.ml`.
Go Claude. My contribution was keeping the explorations looking at relevant
parts of the system, and not disappearing off on sometimes ridiculous and
unbelievable tangents. Maybe it would have got there on its own, but who knows
the tokens required and the GPUs scorched...

Plug that back into my little script. ocamlcommon.cma still different. At this
point, a line from [Four Weddings and a Funeral](https://en.wikipedia.org/wiki/Four_Weddings_and_a_Funeral)
could be heard loud and clear in the human mind. It's the one which follows
"Dear Lord, forgive me for what I am about to, ah, say in this magnificent place
of worship...".

The fix was definitely working. But a quick bit of further experimentation
revealed that including _other_ .mli files before utils/config.ml (and there are
a lot) was causing the information to change.

So:

```
$ claude -c
```

As a human of hopefully normal emotional response to situations, the feeling of
being back at square one would normally have meant I'd have at least needed a
coffee before being able to face dusting off all the tools and scripts which had
been constructed in the previous investigations. But here of course the LLM
doesn't care and was straight into using the tools previously constructed to
look at the revised problem. A lot more `Obj.magic`-like investigations later
looking at the shape of some debugging information, and Claude found another bit
to reset, this time in `Ctype`. All the level information in the type-checker
isn't reset between compilations. Not a semantic issue, because the type checker
uses those numbers _relatively_, but again they leak into the representation of
some of the debugging information.

And it was working 🥳

Next up was trying to put those fixes into something resembling a commit series
that might one day be an acceptable PR. What I really wanted was a test. Claude
was great for this, although it lacks anything approximating taste (and this is
me writing...!). However, with no feelings to be hurt, the pointers were easy to
issue and the results impressive - especially constructing a non-trivial
ocamltest block. The result is previewable in [dra27/ocaml#237](https://github.com/dra27/ocaml/pull/237/files)
on my GitHub fork, and the test is entirely Claude's.

Having got to this stage, I extended the compiler with some of Lucas's patches,
and started passing just the .ml files for compilation, allowing the compiler to
compile the .mli files on demand, as before. With some idle tinkering, I got to
the end of "coreall", which is the point in OCaml's build process where
`ocamlc`, the bytecode versions of everything in `tools/` and `ocamllex` have
all been compiled, along with the Standard Library. That was all being done from
a single compiler process, where the OCaml script driving the compiler consisted
mostly of the list of .ml files. Coupled with the predictive load path I'd
already put together, at this stage the "plumbing" needed in the scheduler is
just:

```ocaml
let compile_file source_file () =
  Compenv.readenv Format.std_formatter (Before_compile source_file);
  let output_prefix = Compenv.output_prefix source_file in
  if Filename.extension source_file = ".mli" then
    Compile.interface ~source_file ~output_prefix
  else
    let start_from = Clflags.Compiler_pass.Parsing in
    Compile.implementation ~start_from ~source_file ~output_prefix

let rec execute task =
  try task ()
  with effect (Load_path.Missing path), k ->
    let file = Filename.chop_extension path ^ ".mli" in
    execute (compile_file file);
    execute (Effect.Deep.continue k)
```

(as an aside, when it goes to being done with Domains I'll possibly switch it to
a shallow handler, because the call stack with the deep handlers isn't as
reasonable as I'd hoped for, but to be honest I just wanted to see it work!)

Fascinatingly, all the artefacts (.cma and binaries) being produced were
identical except for the `Lazy` module in the Standard Library!


```
$ claude -c
```

Claude was simultaneously amazing and useless at this. Amazing, because I was
prompting some of this while cooking a meal, so being able to bark an
instruction (actually, I hadn't set it up for voice - I was just quickly
typing) and then leave it to think for a minute or two was strangely efficient,
because investigating this on my own would have taken too much continuous
concentration. It was useless because we didn't get anywhere near a believable
explanation, despite various efforts at resetting things. Sometimes you just
have to say `/exit` (and eat a meal...).

However, after the aforementioned meal, I dug into it a bit further. The issue
here was clearly to do with some state in the compiler - if ocamlcommon.cma or
ocamlmiddleend.cma were compiled, then the Lazy module differed. Incidentally,
at this point this wasn't debug information which varied, it was the actual
module, but it was still semantically the same. Claude had correctly identified
that it was to do with the marshalling, and we had identified that there was a
difference in string sharing (so not entirely useless, in fairness). I carried
on poking and, with a little bit of jerry-rigging, managed to determine the
relatively small set of files in flambda and in ocamlcommon whose compilation
caused the change in Lazy. I was highly suspicious it was to do with compilation
of lazy values.

```
$ claude -c
```

Feeding this information to Claude was a much better trick - the reasoning at
this point would contradict its own tangents ("I should look at ... but wait,
the user has given me the list of affected files"). Impressively, we did hone in
on the much more complex explanation for this third issue, which is to do with
lazy values used in globals in the `Matching` module. In this particular case,
if the compiler has compiled a file which matched on a lazy, causing
`Matching.code_force_lazy_block` to be forced in the compiler and thus the
`CamlinternalLazy` identified to be added to the current persistent environment,
then a subsequent module (in this case `lazy.ml` in the Standard Library) which
_both_ pattern matches on a lazy and which also refers to `CamlinternalLazy`
ends up with two extern'd string representations of `CamlinternalLazy` instead
of one. The reason is that the forced code block in `Matching` still refers to a
string used in a _previous_ persistent environment. It's not a semantic issue at
all, but it manifests itself because the string is not shared when the
subsequent file looks up the `CamlinternalLazy` identifier.

It was a battle to update the test to show this behaviour, but in fairness that
would have been a battle anyway! However, we got there too.

Three reproducibility issues identified, and a viable PR produced - with tests!
