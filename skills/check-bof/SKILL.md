---
name: check-bof
description: "Audit a BOF extension: cross-reference entry.c, CNA, and extension.json against Sliver's extension system. Reports argument safety, manifest correctness, and documentation quality. Read-only — changes nothing."
argument-hint: "<bof-directory-path>"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Task
  - Bash
---

# Check BOF Extension

You are auditing a BOF (Beacon Object File) extension for safety, correctness, and
documentation quality against the Sliver C2 extension system. **A bad manifest WILL
crash the implant process.** Your job is to produce a thorough report — not to fix
anything. This skill is strictly read-only.

## Setup

### Load Reference
Read `reference/sliver-extension-system.md` in the plugin root.
This contains the baked-in Sliver extension system knowledge.

### Verify Against Live Source (if available)
Use Glob to search for `**/client/command/extensions/load.go` in the workspace.
If found, the Sliver repo is present — verify critical details (argument types in
`checkExtensionArgs`, default handling in `bofApplyDefaultValue`, validation in
`validManifest`) against the live source. Note any discrepancies with the reference
doc in your report.

## Input

The user provides a path to a BOF directory. It may contain any combination of:
- `extension.json` — Sliver extension manifest (may be V1, V2, or absent)
- `entry.c` (or other .c/.h source) — BOF C source code
- A `.cna` file — Cobalt Strike aggressor script (may be in this directory or elsewhere in the repo)
- A `.axs` file — Adaptix C2 script (functionally equivalent to CNA for manifest work;
  see Section 9.4 of the reference doc for syntax mapping)

Search for all of the above. Note what's present and what's missing. When an `.axs`
file is found, apply the same analysis procedures documented for CNA files — the
bof_pack format strings, argument definitions, and help text serve the same purpose.

## Procedure

### Part 1: Inventory

Report what you found:
- **extension.json**: Present (V1/V2) or MISSING
- **Source code**: path to entry.c or equivalent
- **CNA/AXS**: path to .cna or .axs file and the specific command handler within it
- **Compiled .o files**: Do they exist where the manifest references them?
- **Directory layout**: Flat (`.o` files beside manifest) or sub-directory
  (`dist/{os}/{arch}/name.o`). Note which pattern is in use.
- **Platform coverage**: Which OS/arch combos have `.o` files? Windows-only
  (traditional) or cross-platform?

If extension.json is missing, the remaining checks focus on what a manifest SHOULD
contain based on entry.c and CNA analysis. This is valuable prep for `/update-bof`.

### Part 2: Argument Audit

This is the most critical section. Wrong arguments = crashed implant.

#### 2a: Extract from entry.c (THE source of truth)

Find the entrypoint function (usually `go()`). Trace ALL BeaconData* calls in
sequential order:
- `BeaconDataExtract` → string (check context: could be wstring if used with wide APIs)
- `BeaconDataInt` → integer (32-bit)
- `BeaconDataShort` → short (16-bit)
- `BeaconDataExtract` with length used for binary data → file

Build the authoritative argument table:
```
Position | BeaconData Call      | C Type/Variable | Used For
---------|---------------------|-----------------|----------
1        | BeaconDataExtract   | char* hostname  | Target machine
2        | BeaconDataInt       | int pid         | Process ID
```

Watch for:
- **Passthrough pattern**: BOF has exactly one `BeaconDataExtract` call and passes
  the result to an internal parser (e.g., `/key:value` style). See Section 9.5 of
  the reference doc. This is common in Adaptix/Rubeus-style BOFs. The manifest
  needs only one `"string"` argument — do not attempt to decompose the internal
  `/key:value` structure into discrete manifest arguments.
- **Conditional reads**: BOF reads different args based on a flag/mode value.
  Flag this — the manifest may not be able to represent it safely.
- **Loops**: BOF reads a count then loops N times reading more args.
  Flag this — the manifest format has no variadic support.
- **Macros/wrappers**: If BeaconData calls are behind macros or helper functions,
  trace through them. If you can't resolve them, mark as UNCERTAIN.
- **Wide string indicators**: If the extracted string is passed to W-suffix Windows
  APIs (e.g., `CreateProcessW`, `RegOpenKeyExW`), it should be `wstring` in the
  manifest even if extracted with `BeaconDataExtract`.

#### 2b: Extract from CNA (reference, not truth)

Find the command handler in the .cna file. Extract:
- **bof_pack format string** (e.g., `"Zzis"` = wstring, string, int, short)
- **Argument mapping**: Which user args (`$1`, `$2`, etc.) map to which bof_pack positions.
  NOTE: The CNA may reorder args between user input and bof_pack — trace carefully.
- **Optionality**: Look for `iff()` defaults, conditional argument handling, file-open
  dialogs (which imply optional file args)
- **Help text**: The CNA's help/description strings for the command and each argument
- **Validation logic**: Any range checks, enum mappings, or transformations the CNA applies

#### 2c: Extract from extension.json (if present)

Parse the manifest and extract:
- Format (V1 or V2)
- command_name, help, long_help
- `bof_executor`: value or absent (see Section 5.1 of reference doc)
- `depends_on`: value or absent
- Argument list: name, type, optional, default, desc, choices

#### 2d: Cross-Reference

For each argument position, compare all three sources:

| # | entry.c (truth) | CNA bof_pack | extension.json | Status |
|---|----------------|--------------|----------------|--------|
| 1 | BeaconDataExtract → char* hostname | z ($3) | string "hostname" req | OK |
| 2 | BeaconDataInt → DWORD pid | i ($1) | short "pid" req | **TYPE MISMATCH** |

Check:
1. **Count**: Same number of arguments across all sources?
2. **Order**: Manifest array order matches entry.c read order?
   (CNA may reorder between CLI and bof_pack — that's fine as long as
   bof_pack order matches entry.c order)
3. **Types**: Manifest types match entry.c BeaconData calls?
   Use the type mapping table from the reference doc (Section 3.1).
4. **Optional safety**: For args marked optional, what happens when the BOF
   receives the default/zero value? Trace the code path — will it crash?
   (null pointer deref, division by zero, invalid API args, etc.)
5. **Default usability**: For each optional arg without an explicit `default`,
   compare the Sliver zero-value (0 for int/short, "" for string) against
   the CNA's default. If they differ, the BOF will behave differently in
   Sliver than in CS — flag as a usability issue. Also check: does the
   zero-value produce useful behavior, or does it silently do nothing /
   produce garbage output? (e.g., a DNS type of 0, a bitmask of 0 that
   selects nothing, an uninitialized variable from a value that matches
   no branch)

### Part 3: Manifest Validation (if extension.json exists)

Run structural checks:

**Critical (FAIL if wrong):**
- Valid JSON
- V1: `command_name` and `help` non-empty; `files` has entries with valid os/arch/path
- V2: `name` non-empty; `commands` array non-empty; each command has above
- All argument `type` values are valid (`string`, `wstring`, `int`, `integer`, `short`, `file`)
- No path traversal in file paths (`../`)
- `bof_executor` validation:
  - Must be `""` (absent/empty), `"reflektor"`, or `"coff-loader"`
  - If `"coff-loader"`: `depends_on` must be non-empty (Sliver rejects otherwise)
  - Any other value: FAIL — Sliver rejects the manifest
- For BOF (.o) files: `depends_on` should be `coff-loader` for maximum
  compatibility (ensures COFF loader fallback on all implant versions; see
  execution path analysis below). A BOF without `depends_on` is valid if it
  has `bof_executor: "reflektor"`, but has no fallback for older implants.

**Warning (won't crash but should fix):**
- `entrypoint` missing (fragile)
- `long_help` missing (bad UX)
- `default` missing on optional args (uses type zero-value)
- `default` present on required args (ignored, confusing)
- `"int"` used instead of `"integer"` (inconsistent)
- Argument `desc` is empty or just repeats the name
- `.o` file with no `depends_on` and no `bof_executor` — will ONLY work on
  implants with Reflektor (`CapabilityBOFV1`); fails silently on older implants
- `bof_executor: "reflektor"` without `depends_on` — no fallback for older implants
- `files` entries reference paths that don't exist on disk

#### Execution Path Analysis

Based on the `bof_executor` and `depends_on` combination, determine and report
what execution path the manifest will take. Use the routing table from
Section 5.1.1 of the reference doc. Report:
- Which execution mode(s) are possible
- Whether there's a fallback for older implants
- Whether the manifest is cross-platform capable (Reflektor) or Windows-only (COFF loader)

#### Directory Layout Check

Verify the `files` entries against actual files on disk:
- Do the referenced `.o` file paths exist relative to the manifest's directory?
- Is the layout flat (`name.x64.o`) or sub-directory (`dist/os/arch/name.o`)?
- For sub-directory layouts: are all referenced OS/arch directories populated?
- For flat layouts: do the arch-specific filenames follow the `name.{x64,x86}.o`
  convention?

### Part 4: Documentation Quality

Assess the quality of help text and argument metadata. Grade each as
GOOD / ADEQUATE / POOR / MISSING.

**`help` (short description):**
- GOOD: Clear, one sentence, accurately describes the command
- POOR: Vague ("inject into a process"), wrong (copy-pasted from another command),
  or just repeats the command name
- Check against CNA help text — is it accurate? Is the CNA version better?

**`long_help` (detailed description):**
- GOOD: Explains what the tool does, technique/API used, preconditions, caveats
- POOR: Empty string, generic, or copy-pasted
- Mine CNA for material: help text, usage examples, warnings, special behavior notes
- Mine entry.c for material: comments, API calls that reveal technique

**Argument `name` values:**
- GOOD: Lowercase, descriptive, consistent convention (snake_case preferred)
- POOR: camelCase, PascalCase, single letters, or misleading names
  (e.g., "ComputerName" when CNA calls it "domain")
- Cross-reference CNA parameter names and entry.c variable names

**Argument `desc` values:**
- GOOD: Explains what the arg does, valid values/format, default behavior
- POOR: Empty, repeats the name ("computerName: computerName"), or wrong
- Mine CNA help for better descriptions
- For enum-like integers: CNA often has the value mappings (e.g., hive names → numbers)
- For optional args: describe what happens when omitted

**`choices` usage:**
- Should be present for enum-like args where valid values are well-defined
- CNA enum mappings and validation logic reveal what choices should be

## Output Format

```
# BOF Check: <command_name or directory name>

## Inventory
- extension.json: <path> (V1/V2) | MISSING
- Source: <path to entry.c>
- CNA: <path to .cna> → handler: <command name> | NOT FOUND
- Build artifacts: <.o files present/missing>
- Directory layout: flat / sub-directory
- Platform coverage: <OS/arch list>
- bof_executor: <value> | absent
- depends_on: <value> | absent
- Execution path: <COFFLoader / Reflektor / Reflektor+fallback / ERROR>

## Argument Audit

### Source of Truth (entry.c)
| # | BeaconData Call | C Variable | Purpose |
|---|----------------|-----------|---------|
| 1 | ... | ... | ... |

### Cross-Reference
| # | entry.c | CNA | manifest | Status |
|---|---------|-----|----------|--------|
| 1 | ... | ... | ... | OK / MISMATCH / MISSING |

### Argument Issues
- [CRASH RISK] <description>
- [DISCREPANCY] <description>
- [UNCERTAIN] <description>

## Manifest Validation
| Check | Status | Details |
|-------|--------|---------|
| Valid JSON | PASS/FAIL | ... |
| ... | ... | ... |

## Documentation Quality
| Element | Grade | Notes |
|---------|-------|-------|
| help | GOOD/ADEQUATE/POOR/MISSING | ... |
| long_help | ... | ... |
| Arg: <name> name | ... | ... |
| Arg: <name> desc | ... | ... |

### Suggested Improvements
- help: "<suggested text>" (sourced from: CNA / entry.c comments / analysis)
- long_help: "<suggested text>"
- Arg <name>: rename to "<suggestion>", desc: "<suggested text>"

## Summary

### Safety Assessment
- SAFE / UNSAFE / UNCERTAIN
- <explanation>

### What Needs Work
- [ ] <specific item>
- [ ] <specific item>

### Stop Conditions Hit
- <any items that should block /update-bof from proceeding>
```

## Severity Levels

- **CRASH RISK**: Wrong type, wrong order, wrong count, unsafe optional — WILL crash
- **DISCREPANCY**: Sources disagree; one is wrong but truth is identifiable
- **UNCERTAIN**: Cannot determine with confidence; needs human judgment
- **WARNING**: Won't crash but should be fixed (bad UX, inconsistency, etc.)
- **INFO**: Style/convention issues, doc improvements

## Rules

- NEVER suggest code changes or write files. Report only.
- When you cannot determine something with certainty, say UNCERTAIN explicitly.
  Do not guess.
- entry.c is truth. CNA is reference. When they conflict, entry.c wins.
- Always trace through the full BeaconData call sequence. Do not skip args.
- If entry.c uses macros you cannot resolve, flag as UNCERTAIN — do not assume.
- Include suggested documentation improvements (help, long_help, desc) in your
  report — these are suggestions for `/update-bof` to use, not changes you're making.
