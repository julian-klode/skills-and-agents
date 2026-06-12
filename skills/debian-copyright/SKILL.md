---
name: debian-copyright
description: >
  Use when writing, generating, or reviewing debian/copyright or
  debian/copyright.in.d/<crate> files for Debian packages, including vendored
  Rust crates. Covers DEP-5 / machine-readable copyright format (Format 1.0),
  a two-agent actor-critic loop (writer actor + reviewer critic). Trigger
  keywords: debian/copyright, copyright.in.d, DEP-5, vendored crate, Rust
  vendor, copyright stanza, licensecheck, cme check dpkg-copyright, Expat,
  Unlicense, Apache-2.
---

# Debian copyright generation skill

You are an expert Debian developer. This skill governs how to generate and
verify `debian/copyright.in.d/<crate>` fragments (and `debian/copyright`
itself) using a two-agent **actor-critic loop**: the **writer** (actor) drafts
and edits the file, and the **reviewer** (critic) audits it read-only and
returns a verdict. The actor and critic talk directly to each other — the
writer invokes the reviewer, applies the fixes itself, and re-invokes the
reviewer until it passes.

---

## 1. DEP-5 format reference

Full spec: <https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/>

### File structure

A file is made of stanzas separated by blank lines. There are three kinds:

1. **Header stanza** (once, first): `Format`, `Upstream-Name`,
   `Upstream-Contact`, `Source`, `Comment`
2. **Files stanza** (repeatable): `Files`, `Copyright`, `License`, `Comment`
3. **Stand-alone License stanza** (optional, repeatable): `License`, `Comment`

### Key field rules

- `Files:` — whitespace-separated shell-glob patterns rooted at the source
  tree root. `*` and `?` match slashes and leading dots. **Last match wins**
  — put general globs first, specific overrides last.
- `Copyright:` — one or more free-form copyright notices, one per line
  (continuation lines indented by one space). Must contain at least one
  holder. Year ranges are fine (`2015-2024`). Omit years only when genuinely
  absent from all source files and license files. **One line per distinct
  holder**: when the same holder appears with several years (common, since
  licensecheck emits one line per file), collapse them into a single line
  spanning the earliest to latest year — e.g. `2015 Foo`, `2014 Foo` and
  `2018-2020 Foo` become `2014-2020 Foo`. Match holders by the text after the
  year (treating `2015, Foo` and `2015 Foo` as the same); never merge
  different holders, and never invent years to pad a range.
- `License:` — first line is the short name (synopsis); remaining lines
  (indented, blank lines as ` .`) are the full license text or a pointer.
  In a Files stanza the synopsis alone suffices if a stand-alone License
  stanza provides the text.

### License short name mapping (critical)

| What you see in the crate       | DEP-5 short name     |
|---------------------------------|----------------------|
| MIT / MIT License / Expat       | **Expat**            |
| Apache-2.0 / Apache License 2.0 | **Apache-2**         |
| Apache-2.0 OR MIT (SPDX dual)   | **Apache-2 or Expat**|
| Unlicense / UNLICENSE           | **Unlicense**        |
| ISC License                     | **ISC**              |
| BSD-2-Clause                    | **BSD-2-clause**     |
| BSD-3-Clause                    | **BSD-3-clause**     |
| GPL-2.0-only                    | **GPL-2**            |
| GPL-2.0-or-later                | **GPL-2+**           |
| GPL-3.0-only                    | **GPL-3**            |
| GPL-3.0-or-later                | **GPL-3+**           |
| LGPL-2.1-only                   | **LGPL-2.1**         |
| LGPL-2.1-or-later               | **LGPL-2.1+**        |
| CC0-1.0                         | **CC0-1.0**          |
| public-domain (no copyright)    | **public-domain**    |
| Apache-2.0 WITH LLVM-exception  | **Apache-2 with LLVM exception** |

**Never use** `MIT` or `MIT License` as DEP-5 short names — always use
`Expat`. **Prefer** `Apache-2` over `Apache-2.0`: both parse under
`cme check dpkg-copyright` (the project's own `copyright.header` uses
`Apache-2.0`), but `Apache-2` is the convention for the per-crate fragments,
so use it for consistency.

### Grouping stanzas, and `or` vs `and`

Stanzas are grouped by **license**, never by copyright. Files that share a
license belong in one `Files:` stanza (with one `Copyright:` line per holder);
files with different licenses go in separate stanzas. Then pick each stanza's
`License:` expression:

- **`or`** — use when a file itself offers a **choice**, i.e. its SPDX header
  is `A OR B` (e.g. each file says `Apache-2.0 OR MIT` → `Apache-2 or Expat`).
- **Different actual licenses across files → separate `Files:` stanzas**, one
  per license (general glob first, specific path-globs after). **Do not** join
  them with `and`. E.g. most files `Expat`, a vendored `src/third_party/*`
  subtree `Apache-2` → two stanzas, not one `Expat and Apache-2` stanza.
- **Same license, different copyright holders → ONE stanza** with multiple
  `Copyright:` lines. Differing holders never split a stanza.
- **Exception — an entirely vendored subtree may get its own stanza** even at
  the same license. If the crate bundles a self-contained piece of upstream
  third-party code in its own subdirectory (embedded C library, copied-in
  dependency), you may give that subtree its own `Files:` stanza (with a brief
  `Comment:`) for clarity/attribution. This is for a genuinely separate body of
  code — not merely a different author inside the crate's own sources.
- **`and`** — reserve for only two cases:
  1. **Genuine uncertainty about a single file**: the file header and
     `Cargo.toml` disagree and you cannot tell which governs (e.g. Cargo.toml
     says `MIT OR Apache-2.0` but the file header only says "Licensed under the
     MIT license"). Conservatively require both, and always add a `Comment:`:
     `Comment: Cargo.toml declares X, but some source files only state Y`.
  2. **A confirmed true SPDX `AND`** (e.g. `ISC AND OpenSSL`), where the work
     genuinely requires all the licenses at once.
  Never use `and` just because different files carry different licenses — that
  is the separate-stanza case.

### Compound license expressions: use commas, never parentheses

The DEP-5 `License:` synopsis grammar accepted by `cme check dpkg-copyright`
supports the operators `and`, `or`, `and/or` and **commas** for grouping —
but it does **not** support parentheses. A bare `(` or a name glued to a
paren such as `(Apache-2` will be rejected with
*"license '(Apache-2' is not declared in a stand-alone License paragraph"*.

When an SPDX expression needs grouping (mixed `AND`/`OR`), translate the
parentheses into a comma. The comma binds looser than the adjacent operator,
i.e. it terminates the preceding sub-expression:

| SPDX (Cargo.toml)                  | DEP-5 synopsis                       |
|------------------------------------|--------------------------------------|
| `ISC AND (Apache-2.0 OR ISC)`      | `Apache-2 or ISC, and ISC`           |
| `ISC AND (Apache-2.0 OR ISC) AND OpenSSL` | `Apache-2 or ISC, and ISC and OpenSSL` |
| `(MIT OR Apache-2.0) AND Unicode-3.0` | `Expat or Apache-2, and Unicode-3.0` |

Put the parenthesised `or`-group first, close it with a comma, then continue
with the `and`-joined remainder. **Never emit `(` or `)` in a `License:`
field.**

### License exceptions (e.g. LLVM-exception)

SPDX `WITH` exceptions must be written in the exact form the grammar parses:

```
license_exception: 'with' abbrev 'exception'
```

That is: the literal lowercase word `with`, a single whitespace-free token
(the exception abbreviation), then the literal word `exception`. So
`Apache-2.0 WITH LLVM-exception` becomes the short name
**`Apache-2 with LLVM exception`** (lowercase `with`, the hyphen in
`LLVM-exception` removed, a space before the trailing word `exception`).

- Use the **same** short name both in the `Files:` synopsis and in the
  stand-alone `License:` paragraph that carries the text.
- Do **not** hyphenate (`LLVM-exception`), do **not** uppercase (`WITH`), and
  do **not** append anything after `exception` — any of these break the
  grammar.
- Because the name itself is a multi-token operator expression, it can never
  carry a disambiguating `-<suffix>`. If two crates have textually different
  copies of the same exception license, treat them as one license (they
  differ only in wording/whitespace) and keep a single stand-alone stanza.

### License text handling

- **Apache-2**: use the system copy pointer:
  ```
  License: Apache-2
   On Debian systems, the full text of the Apache License Version 2.0
   can be found in the file `/usr/share/common-licenses/Apache-2.0`.
  ```
- **GPL-2, GPL-2+**: pointer to `/usr/share/common-licenses/GPL-2`.
- **GPL-3, GPL-3+**: pointer to `/usr/share/common-licenses/GPL-3`.
- **LGPL-2.1, LGPL-2.1+**: pointer to `/usr/share/common-licenses/LGPL-2.1`.
- **All other licenses** (Expat, ISC, BSD-*, Unlicense, CC0, etc.):
  embed the license text from the crate's license file, with blank
  lines replaced by ` .` (a space then a dot).
  **Strip** any title line (e.g. "The MIT License (MIT)", "ISC License")
  — these are section headers that would prevent deduplication.
  **Strip any real copyright notice** with a specific name and year
  (e.g. "Copyright (c) 2015 Andrew Gallant") — these belong in the
  `Copyright:` field of the `Files:` stanza, not duplicated here.
  **Keep template/placeholder copyright lines** (e.g. "Copyright (C)
  <year> <name of author>" from GPL-3's "How to Apply These Terms"
  instructions) — these are part of the license boilerplate, not
  actual copyright statements for the work.
  You do not need to match wording exactly across crates: the merge step
  (`debian/bin/merge-copyright`) deduplicates stand-alone license texts by
  comparing them with all whitespace collapsed, and the package-wide canonical
  texts already live in `debian/copyright.header`. Identical-after-whitespace
  copies collapse into one stanza automatically.

### Copyright field for UNLICENSE / public-domain files

The Unlicense is a public-domain dedication. The `Copyright:` field for the
`UNLICENSE` file itself should record the dedication statement, e.g.:
```
Copyright: Authors of <crate> (public domain dedication; see UNLICENSE)
```
Do **not** invent a copyright holder for a file that dedicates itself to the
public domain.

### `Source:` field

Always populate from `Cargo.toml`: use `repository` if present, else
`homepage`, else `https://crates.io/crates/<name>`. Never invent a URL.

### Filtered-out stub crates

`cargo-vendor-filterer` excludes crates that aren't needed on the target
platforms by replacing their body with a stub: an **empty `src/lib.rs`**
(0 bytes or whitespace-only) and no other real code — only `Cargo.toml` and
metadata remain. A crate is a stub when it has **no source file** (`.rs`,
`.c`, `.h`, `.cpp`, `.cc`, `.go`, `.py`, …) containing any non-whitespace
content.

Such crates ship no code, so they need **no copyright stanzas**. For a stub,
write an **empty** `debian/copyright.in.d/<crate>` file (zero bytes) and do
nothing else — no licensecheck, no grep, no review loop. The empty file is the
correct, complete result: it records that the crate was handled deliberately
(merge-copyright simply contributes nothing from it) rather than overlooked.

---

## 2. The two agents

The work is split between two subagents whose full operating instructions live
in their own agent definitions. This skill provides the shared DEP-5 reference
(section 1), orchestration (section 3), pitfalls (section 4), and an example
(section 5) that both agents rely on. **Do not duplicate the agents' workflows
here** — edit the agent files when the workflow changes.

### `debian-copyright-writer` (the actor)

**Role**: drafts `debian/copyright.in.d/<crate-name>`, then drives the review
loop until the critic passes. It owns the file end to end — it drafts, has the
critic review, and **applies every fix itself**. Editing is the actor's job.

**Input**: exactly **one** crate directory path (e.g.
`rust-vendor/aho-corasick`). The writer is single-crate; it never loops over
the tree. Batch coordination is the primary agent's responsibility
(section 3).

In outline, the actor: reads `Cargo.toml` and the top-level license files;
runs `licensecheck --merge-licenses -r --deb-machine` **once** as a starting
point (it emits Debian short names and per-file copyright, but its header
stanza and its `Copyright: NONE`/`License: UNKNOWN` rows must be discarded, and
it misses some copyright statements); then **greps the files itself for both
copyright statements and license declarations (SPDX headers)** and reconciles
them with licensecheck, adding any holders or licenses licensecheck missed;
cross-checks against `Cargo.toml` and the license files; writes the DEP-5 file;
then runs the actor-critic loop (invoke critic → apply fixes → re-invoke) for
up to 3 rounds. Full step-by-step instructions are in the
`debian-copyright-writer` agent definition.

### `debian-copyright-reviewer` (the critic)

**Role**: audits the generated file **read-only** and returns a strict verdict.
It never edits the file and never invokes another agent — it only judges; the
actor applies. It runs `cme check dpkg-copyright -file <fragment>` (the same
format gate the package build uses) plus content checks. It is read-only **by
construction** — `edit: deny`, `write: false`, `task: false` make file changes
and agent-spawning impossible — so it is given `bash: allow`. That avoids the
invisible-prompt hang an `ask` default causes inside a subagent and stops
harmless command forms (pipes, `2>&1`, `; echo $?`) from tripping a deny-list,
while the critic still cannot change anything.

**Input**: the path to `debian/copyright.in.d/<crate>` and the crate directory.

**Output**: either `VERDICT: PASS`, or `VERDICT: FAIL` with a numbered list of
issues, each citing the field/line that is wrong, what the source actually says
vs. what was written, and the exact fix required.

Locking the critic read-only is deliberate: it eliminates the risk of the
critic hallucinating formatting errors or truncating the file while trying to
"fix" it. Full checklist is in the `debian-copyright-reviewer` agent
definition.

---

## 3. Orchestration

There are three tiers, and each does exactly one job:

- **Primary agent** — coordinates the batch (this is *not* a subagent). It
  decides which crates to process and invokes the writer once per crate.
- **`debian-copyright-writer` (actor)** — handles **one** crate: drafts the
  file and drives the inner review loop.
- **`debian-copyright-reviewer` (critic)** — read-only; judges one file and
  returns a verdict. Never edits, never calls another agent.

### Per-crate actor-critic loop (inside one writer invocation)

```
1. Primary invokes debian-copyright-writer (actor) on ONE crate.
   The actor then, on its own:
     a. Drafts debian/copyright.in.d/<crate>.
     b. Invokes debian-copyright-reviewer (critic)        → verdict
     c. If FAIL: applies the critic's fixes itself,
        then re-invokes the critic                        → verdict
     d. Repeats (b–c) until PASS (max 3 rounds;
        report unresolved issues to the primary if still failing).
2. The actor reports the file path, verdict, round count, and any licenses it
   deliberately excluded/added — concisely (one crate's worth).
```

### Batch coordination (the primary agent's job)

The writer is single-crate, so the **primary agent** owns iterating the tree.
The `/copyright` command is the named entry point (`/copyright all` for the
whole tree, `/copyright <crate-dir>` for one). For a whole-tree run the primary:

1. **Enumerate** the direct subdirectories of `rust-vendor/`.
2. **Handle stub crates directly — do not spawn a writer for them.** A crate
   is a filtered-out stub if it has **no source file** (`.rs`, `.c`, `.h`,
   `.cpp`, `.cc`, `.go`, `.py`, …) containing any non-whitespace content (the
   typical signal is an empty `src/lib.rs`; see §"Filtered-out stub crates").
   For each stub, the primary **writes the empty `debian/copyright.in.d/<crate>`
   itself** (zero bytes) and removes it from the work list. This avoids ~one
   subagent invocation per stub (often a large fraction of the tree). One-liner
   to detect a stub (prints nothing ⇒ stub):
   ```
   find rust-vendor/<crate> -type f \( -name '*.rs' -o -name '*.c' -o -name '*.h' \
     -o -name '*.cpp' -o -name '*.cc' -o -name '*.go' -o -name '*.py' \) \
     -exec grep -Ilq '[^[:space:]]' {} \; -print
   ```
3. **Resume — skip already-done crates.** A crate is "done" when its
   `debian/copyright.in.d/<crate>` either:
   - exists, is **non-empty**, **and** passes
     `cme check dpkg-copyright -file debian/copyright.in.d/<crate>` (exit 0); or
   - exists and is **empty (0 bytes)** while the crate is a stub (already
     handled in step 2).
   Skip those; only (re)generate crates that are missing, whose non-empty
   fragment fails `cme`, or whose fragment is empty but the crate is **not** a
   stub (an erroneous empty file). This makes re-runs cheap and self-healing.
4. **Fan out in parallel** (real, non-stub, not-yet-done crates only). Invoke
   `debian-copyright-writer` once per remaining crate, issuing several `task`
   calls in a single message so they run concurrently. The primary **chooses
   its own batch width** based on how many crates remain and tool limits (the
   user may also trigger/adjust fan-out manually). Each invocation gets a
   fresh, isolated context — this is the whole reason the writer is
   single-crate. (The writer also re-checks for a stub as a defensive
   fallback, but the coordinator should already have filtered them out.)
5. **Aggregate** results as **one line per crate**
   (`<crate>: <license> — PASS (N rounds)`, or `<crate>: stub — empty file`),
   surfacing full detail only for crates that still fail after 3 rounds. End
   with a tally (e.g. `361 crates: 266 PASS, 91 stubs, 0 escalated, 4 skipped`).
6. **Assemble + validate.** Once all fragments exist and pass, run
   `debian/rules debian/copyright`. That target runs
   `debian/bin/merge-copyright` (deduplicates stand-alone license texts and
   renames clashing short names) and then `cme check dpkg-copyright` on the
   merged file. Report success, or the `cme` error for human follow-up.

The critic never edits and never calls another agent; all editing lives in the
actor, and all batching lives in the primary.

---

## 4. Common pitfalls reference

| Pitfall                                            | Correct behaviour                                    |
|----------------------------------------------------|------------------------------------------------------|
| Generating stanzas for a filtered-out stub crate (empty `src/lib.rs`, no real code) | Write an empty `debian/copyright.in.d/<crate>` and skip all generation |
| Using `MIT` as a DEP-5 short name                  | Use `Expat`                                          |
| Using `Apache-2.0` in a per-crate fragment         | Prefer `Apache-2` (both parse, but `Apache-2` is the fragment convention) |
| Inventing a copyright year not in any file         | Omit the year or use what is in the LICENSE file     |
| Inventing a copyright holder not in any file       | Remove them; flag to user                            |
| `Source: TODO`                                     | Always populate from `Cargo.toml`                    |
| Glob missing subdirectories                        | Use `rust-vendor/foo/*` to cover all depths          |
| Specific globs before general ones                 | General first; specific overrides last (last wins)   |
| Embedding Apache-2 full text                       | Use `/usr/share/common-licenses/Apache-2.0` pointer  |
| Real copyright in License: stanza body                | Move to `Copyright:` field; only keep template placeholders like `<year>` |
| Splitting a stanza because copyright holders differ | Group by license only; merge same-license files into one stanza with multiple `Copyright:` lines |
| An entirely vendored/embedded subtree at the same license | May get its own `Files:` stanza (with a `Comment:`) for attribution — this is allowed, not an error |
| Repeating the same holder with different years     | Collapse to one line with the min–max range (`2014-2020 Foo`) |
| Missing `Copyright:` in a `Files:` stanza          | Every `Files:` stanza requires a `Copyright:` field  |
| Cargo.toml `license` field is SPDX syntax          | Translate to DEP-5 (OR → or, fix short names per table) |
| Joining different per-file licenses with `and`     | Split into separate `Files:` stanzas, one per actual license (not one `A and B` stanza) |
| Unsure which license governs a single file (Cargo.toml vs header disagree) | This is the one case for `and`: require both and add a `Comment:` explaining the uncertainty |
| Parentheses in a `License:` synopsis (`(A or B) and C`) | Use a comma to group: `A or B, and C` (grammar has no parens) |
| `WITH`/hyphen in an exception (`Apache-2 WITH LLVM-exception`) | Use `Apache-2 with LLVM exception` (lowercase `with`, space, word `exception`) |
| Dropping an SPDX `OR` alternative (e.g. file says `Apache-2.0 OR ISC OR MIT`, only `Apache-2 or ISC` written) | Account for every alternative the source declares; add the missing license + stand-alone stanza |
| Critic editing the file or calling another agent   | The critic is read-only; only the actor edits and only the actor drives the loop |
| Writer looping over the whole `rust-vendor/` tree   | The writer is single-crate; the primary agent (or `/copyright all`) coordinates the batch and parallel fan-out |
| Re-running the batch from scratch                   | Resume: skip crates whose fragment exists and passes `cme check`; only (re)generate missing/failing ones |
| Hand-writing a crate-identifying `Comment:` in the header | The merge step discards the header stanza; let it auto-generate `Used by:` comments |
| Emitting stanzas for `Copyright: NONE` / `License: UNKNOWN` rows | Drop them; they are metadata/build files covered by the catch-all glob |
| Trusting licensecheck `Copyright:` lines as complete | licensecheck misses some statements; grep the files yourself (`copyright`, `(c)`, `©`) and add any missing holders |
| Suffixing or "fixing" a license whose text matches another crate's | The merge step deduplicates identical texts and renames clashes automatically |
| Not validating grammar at all                       | The critic runs `cme check dpkg-copyright -file <fragment>` — the real gate |

---

## 5. Example output

For `rust-vendor/aho-corasick` (Unlicense OR MIT, author Andrew Gallant).
Note: the header stanza (`Format`/`Source`) is discarded by
`debian/bin/merge-copyright`; only the `Files:` and stand-alone `License:`
stanzas survive into the final `debian/copyright`. Do **not** hand-write a
`Comment:` identifying the crate — the merge tool generates `Used by: <crate>`
comments on the license stanzas automatically.

```
Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
Source: https://github.com/BurntSushi/aho-corasick

Files: rust-vendor/aho-corasick/*
Copyright: 2015 Andrew Gallant <jamslam@gmail.com>
License: Expat or Unlicense

License: Expat
 Permission is hereby granted, free of charge, to any person obtaining a copy
 of this software and associated documentation files (the "Software"), to deal
 in the Software without restriction, including without limitation the rights
 to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 copies of the Software, and to permit persons to whom the Software is
 furnished to do so, subject to the following conditions:
 .
 The above copyright notice and this permission notice shall be included in
 all copies or substantial portions of the Software.
 .
 THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
 THE SOFTWARE.

License: Unlicense
 This is free and unencumbered software released into the public domain.
 .
 Anyone is free to copy, modify, publish, use, compile, sell, or
 distribute this software, either in source code form or as a compiled
 binary, for any purpose, commercial or non-commercial, and by any
 means.
 .
 In jurisdictions that recognize copyright laws, the author or authors
 of this software dedicate any and all copyright interest in the
 software to the public domain. We make this dedication for the benefit
 of the public at large and to the detriment of our heirs and
 successors. We intend this dedication to be an overt act of
 relinquishment in perpetuity of all present and future rights to this
 software under copyright law.
 .
 THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
 IN NO EVENT SHALL THE AUTHORS BE LIABLE FOR ANY CLAIM, DAMAGES OR
 OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE,
 ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
 OTHER DEALINGS IN THE SOFTWARE.
 .
 For more information, please refer to <http://unlicense.org/>
```
