---
layout: post
title: "Sometimes it's just knowing where to tap"
categories: platform
tags: "ocaml windows"
---

```diff
@@ -44,6 +44,8 @@
 # the lines involved in the conflict, which is arguably worse
 #/Changes                 merge=union

+testsuite                export-ignore
+
 # No header for text and META files (would be too obtrusive).
 *.md                     typo.missing-header
 README*                  typo.missing-header
```

First time users of OCaml on Windows: **25% speedup on switch creation**. All
platforms gain a benefit, even if it's much smaller. As both [rustup](https://www.youtube.com/watch?v=qbKGw8MQ0i8)
and [uv](https://www.youtube.com/watch?v=gSKTfG1GXYQ) have taught us: don't do
stuff you don't need to (uv) and making Windows better usually benefits Linux,
or at least doesn't make it worse (rustup).

PR to follow soon: it turns out it's worth tapping a few more times, but then a
little bit of soldering is needed...
