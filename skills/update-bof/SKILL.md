---
name: update-bof
description: "Update a BOF's extension.json: convert V1→V2, create from scratch, improve documentation, fix argument issues. Runs /check-bof first. Stops on safety uncertainty. Asks before writing."
argument-hint: "<bof-directory-path>"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Task
  - Edit
  - Write
  - Bash
---

# Update BOF Extension

You are updating a BOF (Beacon Object File) extension manifest for the Sliver C2
extension system. **A bad manifest WILL crash the implant process.** You will STOP
and refuse to produce output if you cannot verify the result is safe.

## Setup

### Load Reference
Read `reference/sliver-extension-system.md` in the plugin root.

### Verify Against Live Source (if available)
Use Glob to search for `**/client/command/extensions/load.go`. If found, verify
critical details against live source.

## Input

The user provides a path to a BOF directory. The directory may be in any state:
- No extension.json at all (create from scratch)
- V1 extension.json (convert to V2 + improve)
- V2 extension.json (improve docs, fix issues)
- Any of the above with varying documentation quality

## Mandatory First Step: Run /check-bof

Before making ANY changes, perform the full `/check-bof` audit procedure.
Read the check-bof SKILL.md and follow its complete procedure. Present the
findings to the user.

**STOP CONDITIONS — Do NOT proceed to updates if ANY of these are true:**
- Argument count cannot be determined from entry.c
- Argument order cannot be determined from entry.c
- Argument types cannot be determined from entry.c
- entry.c uses conditional/variadic argument parsing the manifest cannot represent
- entry.c and CNA fundamentally disagree and you cannot determine which is correct
- entry.c cannot be found and no other verification source exists

If a stop condition is hit, present the check-bof report and explain what's blocking.
Ask the user how to proceed. **Do not guess. Do not produce a "best effort" manifest.**

## Update Procedure

After the check passes, determine what work is needed and present a plan to the
user before doing anything. The plan should list exactly what changes you'll make.

### Detect State and Plan

| Current State | Actions |
|---|---|
| No extension.json | Build manifest from scratch using entry.c + CNA |
| V1 manifest | Structural conversion to V2 + arg fixes + doc improvements |
| V2 manifest, arg issues | Fix argument issues + doc improvements |
| V2 manifest, docs only | Documentation improvements only |
| V2 manifest, all clean | Report "nothing to do" |

### Action: Structural Conversion (V1 → V2)

1. **Create the V2 structure:**
   ```json
   {
       "name": "<Human Readable Name (Category)>",
       "package_name": "<existing command_name value>",
       "version": "<keep existing>",
       "extension_author": "<keep existing>",
       "original_author": "<keep existing>",
       "repo_url": "<keep existing or fix if 'N/A'>",
       "commands": [
           {
               <command fields moved here>
           }
       ]
   }
   ```

2. **Move these fields into `commands[0]`:**
   command_name, help, long_help, depends_on, entrypoint, files, arguments

3. **Set top-level fields:**
   - `name`: Human-readable display name. Convention: "Descriptive Name (Remote)"
     or "Descriptive Name (Injection)" — not the command slug
   - `package_name`: Set to the EXISTING `command_name` value. Do not invent
     a new one — this must match what the armory/build system expects.

4. **Preserve exactly:**
   - `command_name` — NEVER rename without explicit user discussion
   - `version` — CI/release workflow sets this
   - `extension_author`, `original_author` — keep as-is
   - `repo_url` — keep as-is unless literally "N/A", then use the sliverarmory GitHub URL

### Action: Build From Scratch (no existing manifest)

When no extension.json exists, build one from entry.c and CNA analysis.
This requires extra caution.

1. **Determine command_name**: Ask the user. Do not invent one — it has
   armory/tarball naming implications.

2. **Build the V2 structure** with all fields populated from analysis:
   - Arguments from entry.c (types, order, count — verified)
   - Help text from CNA and entry.c comments
   - Optionality from CNA behavior (but verify BOF handles defaults safely)

3. **Extra safety gate**: Present the manifest and explicitly ask the user
   to verify the argument list matches their understanding of the BOF.

### Action: Fix Argument Issues

Based on check-bof findings, fix argument problems. For each fix:

- **Type correction** (e.g., `"int"` → `"integer"`, or `"short"` → `"integer"`
  when entry.c uses `BeaconDataInt`): Apply directly — this is a safety fix.

- **Argument order correction**: Apply if entry.c read order is clear.
  This is a safety fix. Note the change prominently in the summary.

- **Optional/default changes**: Only mark an argument optional if BOTH:
  1. The CNA treats it as optional (has default/conditional logic), AND
  2. You've traced the entry.c code path with the default value and it won't crash

  If you're unsure → keep it required. Safer to make the user provide a value
  than to silently pass a crash-causing default.

- **Adding `"default"` values**: For optional args, set defaults based on:
  1. CNA default values (from `iff()` logic or documented defaults)
  2. entry.c behavior with zero/empty values
  3. Type-appropriate zero values as last resort (0 for int/short, "" for string)

  **Always set an explicit `default` when the CNA default differs from the
  Sliver zero-value** (0 for int/short, "" for string). Also set one when the
  zero-value produces useless/broken behavior even if it won't crash (e.g.,
  a bitmask of 0 that selects nothing, a record type of 0 that is invalid).

  For `"file"` type args: do NOT set a default. There is no meaningful file default.

- **Adding `"choices"`**: Add for enum-like args where valid values are clear
  from CNA mappings or entry.c logic. Choices are tab-completion hints only
  (not enforced), so they're low-risk to add.

### Action: Documentation Improvements

Use material gathered during the check-bof phase to improve documentation.
Sources to mine, in priority order:

#### help (short description)
1. CNA help/description string for the command
2. entry.c comments at the top of the file or above the entrypoint
3. Your analysis of what the BOF actually does (based on API calls in entry.c)

Rules:
- One clear sentence, proper grammar
- Accurately describes what THIS command does (watch for copy-paste from other BOFs)
- Don't be vague ("inject into a process") — be specific about the technique/target

#### long_help (detailed description)
Build from multiple sources:
1. CNA help text (often has usage examples, warnings, prerequisites)
2. entry.c: What Windows APIs does it call? What technique does it use?
3. CNA: Any OPSEC notes, warnings, limitations?
4. CNA: What privileges are required? (look for `isadmin()` checks, privilege requests)
5. entry.c: Comments about behavior, edge cases, OS version requirements

Structure:
- What the command does (first sentence — can mirror `help`)
- How it works (technique, APIs used)
- Prerequisites (privileges, target requirements)
- Caveats and warnings (OPSEC notes, known limitations, OS version constraints)

#### Argument names
- Use lowercase with underscores: `computer_name` not `computerName` or `ComputerName`
- Be descriptive: `service_name` not `svc`, `hostname` not `h`
- Match the semantic meaning: if CNA calls it "domain" and entry.c uses it as a
  domain parameter, don't name it "computer_name" in the manifest
- **NOTE**: Argument names become `--flag-names` in Sliver. Choose names that make
  good CLI flags.

#### Argument descriptions (`desc`)
- Explain what the argument controls, not just what it is
- Include valid values/format: `"Registry hive: 0=HKCR, 1=HKCU, 2=HKLM, 3=HKU"`
- Include examples where helpful: `"UNC path to share (e.g., \\\\server\\share)"`
- For optional args: explain what happens when omitted
- Mine CNA for value mappings, format descriptions, examples
- Mine entry.c comments for additional context

### Action: Flag for Discussion

Some issues should be flagged but NOT automatically resolved:
- **Multi-command consolidation**: If related BOFs (e.g., suspend/resume) could
  share a manifest, note it but do not consolidate without user discussion.
- **command_name renaming**: If naming is inconsistent (hyphens vs underscores),
  note it but do not rename. Discuss armory/tarball implications.
- **Build script changes**: If make_bof.sh needs V2 support, note it but do not modify.
- **Missing BOFs**: If a CNA defines a command with no extension.json and no clear
  entry.c, note it but do not create a manifest from guesswork.

## Output

Present to the user:

1. **Check-bof report summary** — Key findings from the audit
2. **Proposed changes** — Exactly what you plan to modify, organized by category:
   - Structural (V1→V2 conversion or new manifest)
   - Safety fixes (type corrections, order fixes)
   - Optional/default improvements
   - Documentation improvements (show before → after for each)
   - Items flagged for discussion
3. **The complete updated extension.json** — Full file content for review
4. **Ask for confirmation** — "Write this to <path>?"

**NEVER overwrite extension.json without explicit user approval.**

## Formatting Rules

- Keep JSON properly formatted with consistent indentation (4 spaces)
- Do NOT flatten argument objects to single lines. Each argument field on its own line:
  ```json
  {
      "name": "hostname",
      "desc": "Target computer hostname or IP address",
      "type": "string",
      "optional": true,
      "default": ""
  }
  ```
- Preserve any existing formatting style where possible (e.g., if the repo
  uses 2-space indent, match it)

## Safety Rules

- NEVER change `command_name` without explicit user approval and discussion.
- NEVER guess about argument types, order, or count. entry.c is truth.
- NEVER mark an argument optional unless you've verified the BOF handles the
  default value safely by tracing the code path in entry.c.
- NEVER produce a manifest you're unsure about. Stop and flag instead.
- NEVER write files without showing the user exactly what you'll write.
- A manifest that might be wrong is WORSE than no manifest at all.
