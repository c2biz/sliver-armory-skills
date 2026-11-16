# BOF Extension Manifest Workspace

## Safety Principle

A bad manifest crashes the implant process on the target. When uncertain, **STOP
and flag** — never guess and ship. It is better to not produce a manifest than to
produce one that might crash someone's process.

## Sources of Truth (in order)

1. **Sliver source code** — How the extension system actually works (parsing, validation,
   argument packing, execution). This is THE specification.
2. **BOF source code (entry.c)** — What the BOF actually reads. Argument count, order,
   and types are determined by BeaconDataExtract/BeaconDataInt/BeaconDataShort calls.
3. **CNA / AXS files** — Reference for intended UX, optionality, help text, defaults.
   CNA is Cobalt Strike's format, AXS is Adaptix C2's equivalent. Both serve the same
   purpose for manifest work. Useful but can be wrong. When CNA/AXS and entry.c
   disagree, entry.c wins.
4. **Existing extension.json** — Starting point. May have errors, copy-paste bugs,
   wrong types, or missing arguments.

## Workspace Contents

- `sliver/` — Sliver source repo (may or may not be present)
- Other directories — BOF repos containing extension.json, entry.c, .cna files
  (varies per session)

## Reference Documentation

Sliver extension system internals (manifest formats, argument parsing, binary packing,
naming constraints, crash scenarios) are documented in:
- `reference/sliver-extension-system.md`

Read this before doing any manifest work. When the sliver repo is present, verify
critical details against live source.

## Available Skills

- `/check-bof <bof-dir>` — Full audit: cross-references entry.c + CNA + extension.json
  against Sliver's rules. Covers argument safety, manifest validation, and documentation
  quality. Read-only — changes nothing. Always run this first.
- `/update-bof <bof-dir>` — Does whatever's needed: create manifest from scratch,
  convert V1→V2, fix argument issues, improve docs. Runs /check-bof first (mandatory).
  Stops on safety uncertainty. Shows changes and asks before writing.

## BOF Executor Stance

Sliver now supports two BOF execution engines: the legacy external COFF loader
(`depends_on: "coff-loader"`) and the built-in Reflektor (`bof_executor: "reflektor"`).
The `bof_executor` field in extension.json controls routing.

**This workspace defaults to the COFF loader path.** Always set `depends_on: "coff-loader"`
for BOF commands. Do not add `bof_executor` unless the user explicitly requests it.
Manifests with only `depends_on` work on all implant versions.

If dual-executor support is requested, set both `"depends_on": "coff-loader"` and
`"bof_executor": "reflektor"` — this prefers Reflektor on capable implants with
COFF loader fallback. Never produce reflektor-only manifests (no `depends_on`)
without explicit user approval.

Some repos have build pipelines that inject `bof_executor` at packaging time
(e.g., CS-Situational-Awareness-BOF's `armory_packages.py`). For those repos,
source manifests should NOT include `bof_executor`.

## Key Rules

- **Argument order in the manifest must match the order the BOF reads them.** Wrong
  order = the BOF reads garbage = crash.
- **Argument types must match the BOF's BeaconData calls.** Wrong type = wrong number
  of bytes consumed = cascading corruption = crash.
- **Optional args are still written to the buffer** with their default/zero value. The
  BOF must handle that value safely. Don't mark args optional unless verified.
- **Do NOT rename command_name** without discussing armory/tarball naming implications.
- **Do NOT auto-consolidate** BOFs into multi-command manifests — flag for discussion.
- **Do NOT auto-update** build scripts (make_bof.sh) — flag for discussion.
- **Do NOT add `bof_executor`** unless the user explicitly requests it.
