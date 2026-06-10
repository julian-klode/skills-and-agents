---
name: debian-copyright
description: >
  Use when writing, generating, or reviewing debian/copyright or
  debian/copyright.in.d/<crate> files for Debian packages, including vendored
  Rust crates. Covers DEP-5 / machine-readable copyright format (Format 1.0),
  multi-agent writer + reviewer + editor pipeline. Trigger keywords:
  debian/copyright, copyright.in.d, DEP-5, vendored crate, Rust vendor,
  copyright stanza, decopy, Expat, Unlicense, Apache-2.
---

# Debian copyright generation skill

You are an expert Debian developer. This skill governs how to generate and
verify `debian/copyright.in.d/<crate>` fragments (and `debian/copyright`
itself) using a three-agent pipeline: **writer → reviewer → editor**.

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

**Never use** `MIT`, `Apache-2.0`, `MIT License` etc. as DEP-5 short names.

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
  embed the full license text verbatim from the crate's license file,
  with blank lines replaced by ` .` (a space then a dot).

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

## 2. Agent: `debian-copyright-writer`

**Mode**: subagent
**Input**: one crate directory path (e.g. `rust-vendor/aho-corasick`) or
`rust-vendor/` for batch mode over all crates.
**Output**: writes `debian/copyright.in.d/<crate-name>` for each crate.

### Writer workflow

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

4. **Identify groupings** — determine whether all files share one
   copyright+license, or whether subdirectories/specific files differ. Common
   cases in Rust vendor crates:
   - All files: one author, one dual license → single `Files: rust-vendor/<crate>/*` stanza
   - Embedded third-party code: a subtree with different copyright → separate stanza
   - **Never** create `Files:` stanzas for `LICENSE*`, `COPYING*`, or
     `UNLICENSE*` files. They are already covered by the catch-all
     `rust-vendor/<crate>/*` glob. Their content is captured through
     stand-alone `License:` stanzas.

5. **Write the file** — produce `debian/copyright.in.d/<crate-name>` with:
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

6. **Copyright year sourcing** — years come from (in order of preference):
   a. Explicit copyright lines in source files
   b. The year in a `LICENSE-MIT` / `LICENSE-Apache` etc. file
   c. The `authors` field in `Cargo.toml` if it contains a year
   d. Omit the year only if genuinely unavailable in any of the above

7. **Batch mode** — iterate over every direct subdirectory of `rust-vendor/`
   and process each one. Report progress crate by crate.

---

## 3. Agent: `debian-copyright-reviewer`

**Mode**: subagent
**Input**: path to `debian/copyright.in.d/<crate>` and the crate directory.
**Output**: a structured review report. Either `VERDICT: PASS` or
`VERDICT: FAIL` with a numbered list of issues, each citing:
- The field and line in `debian/copyright.in.d/<crate>` that is wrong
- What was found in the actual source vs. what was written
- The exact fix required

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

6. **Check completeness** — if source files contain copyright lines that are
   not reflected anywhere in the generated stanzas, flag them as missing.

7. **Output the verdict**:
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

## 4. Agent: `debian-copyright-editor`

**Mode**: subagent
**Input**: the reviewer's `VERDICT: FAIL` report and the path to
`debian/copyright.in.d/<crate>`.
**Output**: the file edited in place; a summary of changes made.

### Editor workflow

1. Read the reviewer's issue list carefully.
2. For each issue, make the **minimal targeted edit** to the file:
   - Fix wrong copyright holder names to match source files exactly.
   - Fix wrong license short names.
   - Remove hallucinated copyright holders.
   - Add missing copyright holders found in source files.
   - Fix wrong `Source:` URL to match `Cargo.toml`.
   - Fix wrong license text to match the crate's actual license file.
3. **Never add information not found in the crate's actual files.**
4. After all edits are made, output a summary of every change.
5. **Trigger the reviewer agent again** on the updated file to confirm all
   issues are resolved. Repeat editor→reviewer until `VERDICT: PASS`.

---

## 5. Primary agent invocation workflow

When the user asks to generate or fix a `debian/copyright.in.d` file, the
primary agent orchestrates the pipeline:

```
1. Run debian-copyright-writer  →  file written
2. Run debian-copyright-reviewer  →  verdict
3. If FAIL:
     Run debian-copyright-editor  →  file edited
     Run debian-copyright-reviewer  →  verdict
     Repeat until PASS (max 3 rounds; escalate to user if still failing)
4. Report final status and file path to user.
```

For **batch mode** (whole `rust-vendor/` directory):
- Run the writer over all crates first.
- Then run the reviewer over each crate in sequence.
- Collect all failures, run the editor in one pass per failing crate, then
  re-review. This avoids excessive context switching.

---

## 6. Common pitfalls reference

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
| Splitting stanzas with identical copyright+license | Merge into one stanza with multiple `Copyright:` lines |
| Missing `Copyright:` in a `Files:` stanza          | Every `Files:` stanza requires a `Copyright:` field  |
| Cargo.toml `license` field is SPDX syntax          | Translate to DEP-5 (OR → or, fix short names per table) |

---

## 7. Example output

For `rust-vendor/aho-corasick` (Unlicense OR MIT, author Andrew Gallant):

```
Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
Source: https://github.com/BurntSushi/aho-corasick
Comment: *** only: rust-vendor/aho-corasick ***

Files: rust-vendor/aho-corasick/*
Copyright: 2015 Andrew Gallant <jamslam@gmail.com>
License: Expat or Unlicense

License: Expat
 The MIT License (MIT)
 .
 Copyright (c) 2015 Andrew Gallant
 .
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
