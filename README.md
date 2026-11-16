# Sliver Armory Skills

Claude Code skills for working with [Sliver](https://github.com/BishopFox/sliver)
armory extensions — currently focused on BOF (Beacon Object File) manifest
auditing and generation, with DLL extensions and SDK scaffolding on the roadmap.

A bad manifest crashes the implant process on the target. These skills
cross-reference `entry.c`, CNA scripts, and `extension.json` against Sliver's
extension system to catch argument mismatches before they cause damage.

## Skills

### `/check-bof <bof-dir>`

Read-only audit. Cross-references the BOF source, CNA, and manifest to report:

- Argument type, order, and count mismatches (crash risks)
- Manifest structural issues (invalid JSON, missing fields, bad paths)
- Documentation quality (help text, argument descriptions, naming)

### `/update-bof <bof-dir>`

Creates, converts, or fixes `extension.json` manifests:

- Build a V2 manifest from scratch using `entry.c` + CNA analysis
- Convert V1 manifests to V2
- Fix argument type/order/optionality issues
- Improve help text and argument documentation

Always runs `/check-bof` first. Stops on safety uncertainty. Shows changes and
asks before writing.

## Install

### From Marketplace

```
/plugin marketplace add c2biz/sliver-armory-skills
/plugin install sliver-armory-skills
```

### Manual — Persistent (Local Marketplace)

Register the repo as a local marketplace, then install the plugin from it:

```
/plugin marketplace add /path/to/sliver-armory-skills
/plugin install sliver-armory-skills@sliver-armory-skills
```

The install command opens a panel where you pick scope (user, project, or
local). Once installed, the skills load automatically in every new session.

Or from the shell:

```bash
claude plugin install sliver-armory-skills@sliver-armory-skills --scope user
```

### Manual — Single Session

Load the plugin for one session only, no install needed:

```bash
claude --plugin-dir /path/to/sliver-armory-skills
```

## Usage

Set up a workspace with your BOF repos and (optionally) the Sliver source:

```
~/tools/
  sliver/              # optional — enables live source verification
  SA/nanodump/         # BOF repo with entry.c, .cna, extension.json
  trustedsec/CS-Remote-OPs-BOF/
```

Start Claude in the workspace root and run:

```
/check-bof SA/nanodump
/update-bof SA/nanodump
```

Having `sliver/` in the workspace is recommended but not required. The skills
include a baked-in reference doc (`reference/sliver-extension-system.md`) derived
from Sliver source analysis. When the repo is present, the skills verify critical
details against live source.

## How It Works

The skills follow a strict source-of-truth hierarchy:

1. **Sliver source code** — how the extension system actually parses and executes
2. **`entry.c`** — what the BOF actually reads (argument count, order, types)
3. **CNA files** — intended UX, optionality, help text, defaults
4. **`extension.json`** — starting point, may have errors

When sources disagree, `entry.c` wins. When anything is uncertain, the skills
stop and flag rather than guess.
