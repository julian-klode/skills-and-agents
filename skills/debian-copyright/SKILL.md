---
name: debian-copyright
description: >
  Use when writing, generating, or reviewing debian/copyright or
  debian/copyright.in.d/<crate> files for Debian packages, including vendored
  Rust crates. Covers DEP-5 / machine-readable copyright format (Format 1.0),
  a two-agent actor-critic loop (writer actor + reviewer critic). Trigger
  keywords: debian/copyright, copyright.in.d, DEP-5, vendored crate, Rust
  vendor, copyright stanza, decopy, Expat, Unlicense, Apache-2.
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
  absent from all source files and license files.
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

**Never use** `MIT`, `Apache-2.0`, `MIT License` etc. as DEP-5 short names.

### `or` vs `and` for multi-license stanzas

When a crate has multiple licenses, you must verify source file headers
before choosing `or` vs `and`:

- Use **`or`** only when **every** source file in the stanza's scope
  declares compatibility with **all** listed licenses (e.g. each file
  header says "MIT OR Apache-2.0").
- Use **`and`** (conservative) when some source files only declare a
  subset of the licenses (e.g. file header says "Licensed under the MIT
  license" but Cargo.toml says "MIT OR Apache-2.0"). This means the
  collection requires both because individual files may not be usable
  under the other license.
- When using `and` due to inconsistency, always add a `Comment:` field:
  `Comment: Cargo.toml declares X, but some source files only state Y`

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

---

## 2. Agent: `debian-copyright-writer` (the actor)

**Role**: actor in the actor-critic loop.
**Mode**: subagent
**Tools**: `read`, `write`, `edit`, `bash`, `grep`, `glob`, `task` (so it can
invoke the critic).
**Input**: one crate directory path (e.g. `rust-vendor/aho-corasick`) or
`rust-vendor/` for batch mode over all crates.
**Output**: writes `debian/copyright.in.d/<crate-name>` for each crate, then
drives the review loop until the critic passes.

The writer owns the file end to end: it drafts the file, has it critiqued, and
**applies every fix itself**. Editing is the actor's job.

### Writer workflow — Phase 1: draft

For each crate directory:

1. **Read `Cargo.toml`** — extract:
   - `name` (the crate name; use as the output filename)
   - `version`
   - `authors` (list of `"Name <email>"` strings)
   - `license` (SPDX expression, e.g. `"MIT OR Apache-2.0"`)
   - `repository` and/or `homepage` (for `Source:`)

2. **Read all top-level license/copying files** — read the actual text of
   every file matching `LICENSE*`, `COPYING*`, `UNLICENSE*` at the crate
   root. Identify each file's license by its text content, not just its name.

3. **Scan source files for copyright headers** — grep every `.rs`, `.c`,
   `.h`, `.py`, `.go`, and similar source file (not binaries, not
   `.cargo-checksum.json`) for lines matching `[Cc]opyright`, `©`, or `\(c\)`.
   Collect the raw copyright lines grouped by the subdirectory or file set
   they belong to.

4. **Scan source files for explicit license declarations** — grep source
   files for lines matching "licensed under", "License", "license",
   "Apache", "MIT", "dual-license" and similar phrases in comment
   headers. Note which licenses each file actually declares. This is
   critical for choosing `or` vs `and`.

5. **Cross-check with `licensecheck`** — run licensecheck recursively over
   the crate and reconcile its findings with steps 2-4. This catches
   licenses that manual inspection misses (embedded third-party code,
   OpenSSL/SSLeay dual licenses, files lacking a top-level LICENSE).
   ```
   licensecheck -r --shortname-scheme=debian,spdx rust-vendor/<crate>
   licensecheck -r rust-vendor/<crate> | sed 's/^[^:]*: //' | sort | uniq -c | sort -rn
   ```
   - **Every license licensecheck reports must be accounted for** — in the
     broad `Files:` license expression or in a specific `Files:` stanza for
     the subtree where it appears (use the per-file output to locate it).
   - Map verbose names to DEP-5 short names ("Apache License 2.0" →
     `Apache-2`, "MIT License" → `Expat`, "OpenSSL License" → `OpenSSL`,
     "SSLeay" → `SSLeay`); licensecheck's " and/or " means `or`.
   - licensecheck has false positives (`UNKNOWN`, `*No copyright*`,
     `[generated file]`) and false negatives — investigate, but verify
     against the actual file before adding or dropping a license. Do not
     blindly copy its output.

6. **Identify groupings** — determine whether all files share one
   copyright+license, or whether subdirectories/specific files differ. Common
   cases in Rust vendor crates:
   - All files: one author, one dual license → single `Files: rust-vendor/<crate>/*` stanza
   - Embedded third-party code: a subtree with different copyright → separate stanza
   - **Never** create `Files:` stanzas for `LICENSE*`, `COPYING*`, or
     `UNLICENSE*` files. They are already covered by the catch-all
     `rust-vendor/<crate>/*` glob. Their content is captured through
     stand-alone `License:` stanzas.
   - **Choose `or` vs `and`**: use `or` only when every source file
     in the stanza's scope declares compatibility with all listed
     licenses. Otherwise use `and` with a `Comment:` explaining the
     inconsistency between Cargo.toml and source file headers.

7. **Write the file** — produce `debian/copyright.in.d/<crate-name>` with:
   ```
   Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
   Source: <url from Cargo.toml>
   Comment: *** only: rust-vendor/<crate-name> ***

   Files: rust-vendor/<crate-name>/*
   Copyright: <year(s)> <holder(s) from license files or authors field>
   License: <DEP-5 short name>

   [additional Files: stanzas for specific files/subdirs if needed]

   License: <short-name>
    <full text or system pointer>
   ```

8. **Copyright year sourcing** — years come from (in order of preference):
   a. Explicit copyright lines in source files
   b. The year in a `LICENSE-MIT` / `LICENSE-Apache` etc. file
   c. The `authors` field in `Cargo.toml` if it contains a year
   d. Omit the year only if genuinely unavailable in any of the above

9. **Batch mode** — iterate over every direct subdirectory of `rust-vendor/`
   and process each one. Run the full draft + actor-critic loop for each crate
   before moving to the next. Report progress crate by crate.

### Writer workflow — Phase 2: the actor-critic loop

After writing the draft, the writer (actor) **does not stop**. It runs this
loop, talking directly to the reviewer (critic):

1. **Invoke the critic** via the `task` tool, calling the
   `debian-copyright-reviewer` agent with the path to the file just written and
   the crate directory path. Ask for its `VERDICT`.

2. **Read the verdict.**
   - `VERDICT: PASS` → the loop is done; go to "Finishing" below.
   - `VERDICT: FAIL` → the critic returns a numbered list of issues, each
     citing the field/line, what is wrong, and the exact fix. Continue.

3. **Apply every fix yourself** (the actor edits; the critic never does),
   making the **minimal targeted edit** per issue, using only information
   found in the crate's actual files:
   - Wrong copyright holder → replace with exact text from the source/license
     file.
   - Hallucinated copyright holder → remove that line entirely.
   - Missing copyright holder → add the `Copyright:` line, verbatim from the
     source file where it was found.
   - Wrong license short name → replace with the correct DEP-5 short name.
   - Missing license (declared in source but absent) → add it to the relevant
     `Files:` stanza's `License:` expression and add the matching stand-alone
     `License:` stanza, or create a dedicated `Files:` stanza for the subtree
     that carries it — whichever most accurately reflects the source.
   - Wrong `Source:` URL → replace with the URL from `Cargo.toml`.
   - Wrong/generic license text → replace with verbatim text from the crate's
     actual license file; fix continuation-line indentation (exactly one
     leading space per body line; blank lines as ` .`).
   - `or`/`and` errors, parenthesised synopses, malformed `WITH` exceptions →
     fix per the grammar rules in section 1.

   **Never add information not present in the crate's actual files.** If a fix
   would require inventing information, add a `Comment:` noting the issue and
   flag it in the final report for human review.

4. **Re-invoke the critic** on the updated file (back to step 1).

5. **Loop control** — repeat edit→review up to **3 total review rounds**. If
   the critic still returns `VERDICT: FAIL` after 3 rounds, stop and escalate
   the remaining issues to the user. Never loop forever; never silence the
   critic by deleting legitimate content.

### Finishing

Once the critic returns `VERDICT: PASS` (or 3 rounds are exhausted), report:
- The path of each file written.
- A one-line copyright + license summary per crate.
- Any license licensecheck reported but you deliberately excluded, and why.
- The final verdict, the number of review rounds, and a short summary of the
  changes the critic prompted.

---

## 3. Agent: `debian-copyright-reviewer` (the critic)

**Role**: critic in the actor-critic loop.
**Mode**: subagent
**Tools**: `read`, `bash`, `grep`, `glob` only — **`write: false`,
`edit: false`, `task: false`**. The critic is strictly read-only: it never
edits the file and never invokes another agent. Its `bash` permission is
narrowed to inspection commands (`licensecheck`, `cme`, `rg`/`grep`, `sed`,
`sort`, `uniq`, `head`, `cat`, `ls`, `find`).
**Input**: path to `debian/copyright.in.d/<crate>` and the crate directory.
**Output**: a structured review report the actor can act on directly. Either
`VERDICT: PASS` or `VERDICT: FAIL` with a numbered list of issues, each citing:
- The field and line in `debian/copyright.in.d/<crate>` that is wrong
- What was found in the actual source vs. what was written
- The exact fix required

Locking the critic read-only is deliberate: it eliminates the risk of the
critic hallucinating formatting errors or truncating the file while trying to
"fix" it. The critic only judges; the actor applies.

### Reviewer workflow

1. **Re-read the generated file** in full.

2. **For every `Files:` stanza**:
   - Verify the glob pattern(s) actually match real paths in the crate.
   - Verify each `Copyright:` line names a holder that appears in a license
     file or source file within that glob's scope. Accept years from
     `LICENSE*`/`COPYING*`/`UNLICENSE*` files as authoritative without
     requiring them to appear in every source file.
   - Verify the `License:` short name matches the actual text of the
     corresponding license file. Flag wrong short names (e.g. `MIT` instead
     of `Expat`).

3. **For every stand-alone `License:` stanza**:
   - If it contains a full text: verify the text matches the crate's actual
     license file (not a generic copy-paste that differs from what is present).
   - If it contains a system pointer: verify the license is indeed one that
     has a system copy (`Apache-2`, `GPL-2`, `GPL-3`, `LGPL-2.1`, etc.).

4. **Check `Source:`** — verify the URL is present in `Cargo.toml`
   (`repository` or `homepage`). Flag if it differs.

5. **Check for hallucinated holders** — if a `Copyright:` line names someone
   not found anywhere in the crate's files or `Cargo.toml` `authors`, flag it
   as hallucinated.

6. **Check completeness** — if source files declare a copyright holder or a
   license (including any alternative in an SPDX `OR` expression) that is not
   reflected anywhere in the generated stanzas, flag it as missing. SPDX
   headers are authoritative even when licensecheck collapses an `OR`
   expression to one license; cross-check with `licensecheck` but verify the
   actual file before flagging (ignore false positives such as `UNKNOWN`,
   `*No copyright*`, `[generated file]`, and OpenSSL advertising clauses
   misread as `Apache-1.0`).

7. **Output the verdict** — the actor applies the fixes, so make each issue
   self-contained and actionable:
   ```
   VERDICT: PASS
   ```
   or:
   ```
   VERDICT: FAIL
   Issues:
   1. [field/line] "[wrong text]" — [reason] — fix: [exact corrected text]
   2. ...
   ```

---

## 4. Orchestration: the actor-critic loop

The actor and critic talk directly to each other; the writer (actor) drives the
loop. A coordinating primary agent only kicks it off and reports the result.

```
1. Invoke debian-copyright-writer (actor) on the crate.
   The actor then, on its own:
     a. Drafts debian/copyright.in.d/<crate>.
     b. Invokes debian-copyright-reviewer (critic)        → verdict
     c. If FAIL: applies the critic's fixes itself,
        then re-invokes the critic                        → verdict
     d. Repeats (b–c) until PASS (max 3 rounds;
        escalate to the user if still failing).
2. The actor reports the final file path(s), the verdict, the round count,
   and any licenses it deliberately excluded.
```

The critic never edits and never calls another agent; all editing lives in the
actor.

For **batch mode** (whole `rust-vendor/` directory): the actor runs the full
draft + loop for each crate before moving to the next, reporting progress
crate by crate.

---

## 5. Common pitfalls reference

| Pitfall                                            | Correct behaviour                                    |
|----------------------------------------------------|------------------------------------------------------|
| Using `MIT` as a DEP-5 short name                  | Use `Expat`                                          |
| Using `Apache-2.0`                                 | Use `Apache-2`                                       |
| Inventing a copyright year not in any file         | Omit the year or use what is in the LICENSE file     |
| Inventing a copyright holder not in any file       | Remove them; flag to user                            |
| `Source: TODO`                                     | Always populate from `Cargo.toml`                    |
| Glob missing subdirectories                        | Use `rust-vendor/foo/*` to cover all depths          |
| Specific globs before general ones                 | General first; specific overrides last (last wins)   |
| Embedding Apache-2 full text                       | Use `/usr/share/common-licenses/Apache-2.0` pointer  |
| Real copyright in License: stanza body                | Move to `Copyright:` field; only keep template placeholders like `<year>` |
| Splitting stanzas with identical copyright+license | Merge into one stanza with multiple `Copyright:` lines |
| Missing `Copyright:` in a `Files:` stanza          | Every `Files:` stanza requires a `Copyright:` field  |
| Cargo.toml `license` field is SPDX syntax          | Translate to DEP-5 (OR → or, fix short names per table) |
| `or` used when source files are more restrictive     | Use `and` instead and add `Comment:` explaining the inconsistency |
| Parentheses in a `License:` synopsis (`(A or B) and C`) | Use a comma to group: `A or B, and C` (grammar has no parens) |
| `WITH`/hyphen in an exception (`Apache-2 WITH LLVM-exception`) | Use `Apache-2 with LLVM exception` (lowercase `with`, space, word `exception`) |
| Dropping an SPDX `OR` alternative (e.g. file says `Apache-2.0 OR ISC OR MIT`, only `Apache-2 or ISC` written) | Account for every alternative the source declares; add the missing license + stand-alone stanza |
| Critic editing the file or calling another agent   | The critic is read-only; only the actor edits and only the actor drives the loop |

---

## 6. Example output

For `rust-vendor/aho-corasick` (Unlicense OR MIT, author Andrew Gallant):

```
Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
Source: https://github.com/BurntSushi/aho-corasick
Comment: *** only: rust-vendor/aho-corasick ***

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
