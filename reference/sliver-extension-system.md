# Sliver Extension System Reference

Comprehensive reference for how Sliver loads, parses, validates, and executes
BOF extensions. Derived from Sliver source code analysis. This document IS the
baked-in knowledge; when the Sliver repo is available in the workspace, verify
critical details against live source.

## Source File Map

When the sliver repo is present, these are the key files:

| File | What It Controls |
|------|-----------------|
| `client/command/extensions/load.go` | Manifest structs, parsing, v1→v2 conversion, command registration, execution |
| `client/command/extensions/argparser.go` | BOF argument parsing (flag-style `--name value`) and default handling |
| `client/command/extensions/install.go` | Extension installation from directory or tarball |
| `client/command/extensions/remove.go` | Extension removal by manifest name or command name |
| `client/command/armory/parsers.go` | Armory download logic (tarball/sig filename construction) |
| `client/assets/extensions.go` | Extension discovery on disk |
| `client/core/bof.go` | BOFArgsBuffer — binary argument packing sent to the implant |

---

## 1. Manifest Formats

### 1.1 V1 — Flat, Single-Command (ExtensionManifest_)

Go struct (load.go):
```go
type ExtensionManifest_ struct {
    Name            string               `json:"name"`
    CommandName     string               `json:"command_name"`
    Version         string               `json:"version"`
    ExtensionAuthor string               `json:"extension_author"`
    OriginalAuthor  string               `json:"original_author"`
    RepoURL         string               `json:"repo_url"`
    Help            string               `json:"help"`
    LongHelp        string               `json:"long_help"`
    Files           []*extensionFile     `json:"files"`
    Arguments       []*extensionArgument `json:"arguments"`
    Entrypoint      string               `json:"entrypoint"`
    DependsOn       string               `json:"depends_on"`
    Init            string               `json:"init"`
    RootPath        string               `json:"-"`
}
```

Everything is top-level. One manifest = one command. The `command_name`, `help`,
`files`, `arguments`, `entrypoint`, `depends_on`, and `init` all sit at the top
level alongside metadata.

### 1.2 V2 — Nested, Multi-Command (ExtensionManifest + ExtCommand)

Go structs (load.go):
```go
type ExtensionManifest struct {
    Name            string       `json:"name"`
    PackageName     string       `json:"package_name"`
    Version         string       `json:"version"`
    ExtensionAuthor string       `json:"extension_author"`
    OriginalAuthor  string       `json:"original_author"`
    RepoURL         string       `json:"repo_url"`
    ExtCommand      []*ExtCommand `json:"commands"`
    RootPath        string       `json:"-"`
    ArmoryName      string       `json:"-"`
    ArmoryPK        string       `json:"-"`
}

type ExtCommand struct {
    CommandName string                 `json:"command_name"`
    Help        string                 `json:"help"`
    LongHelp    string                 `json:"long_help"`
    Files       []*extensionFile       `json:"files"`
    Arguments   []*extensionArgument   `json:"arguments"`
    Entrypoint  string                 `json:"entrypoint"`
    DependsOn   string                 `json:"depends_on"`
    Init        string                 `json:"init"`
    Schema      *packages.OutputSchema `json:"schema"`
    Manifest    *ExtensionManifest     // back-reference, not serialized
}
```

Metadata at top level, command(s) in `"commands"` array. New fields:
- `PackageName` — used for install directory naming
- `Schema` — per-command structured output formatting
- `ArmoryName`, `ArmoryPK` — runtime-only, not serialized

### 1.3 Supporting Types

```go
type extensionFile struct {
    OS   string `json:"os"`
    Arch string `json:"arch"`
    Path string `json:"path"`
}

type extensionArgument struct {
    Name     string      `json:"name"`
    Type     string      `json:"type"`
    Desc     string      `json:"desc"`
    Optional bool        `json:"optional"`
    Default  interface{} `json:"default,omitempty"`
    Choices  []string    `json:"choices,omitempty"`
}
```

---

## 2. Manifest Parsing and V1→V2 Conversion

### 2.1 ParseExtensionManifest (load.go)

Detection logic:
1. Try to unmarshal as V2 (`ExtensionManifest`)
2. If that fails OR if `len(ExtCommand) == 0`, try V1 (`ExtensionManifest_`)
3. If V1 succeeds, convert via `convertOldManifest()`
4. Set back-references and initialize schemas
5. Run `validManifest()`

The detection heuristic: V1 manifests have no `"commands"` key, so `ExtCommand`
will always be empty/nil when parsed with the V2 struct.

### 2.2 convertOldManifest (load.go)

Critical detail: **The old manifest's `command_name` becomes the new manifest's `Name`**
(not the old `name` field). The old `name` field is effectively discarded. The comment
says "treating old command name as the manifest name to avoid weird chars mostly."

```go
func convertOldManifest(old *ExtensionManifest_) *ExtensionManifest {
    ret := &ExtensionManifest{
        Name: old.CommandName,  // <-- command_name, NOT name
        // ...
        ExtCommand: []*ExtCommand{{
            CommandName: old.CommandName,
            // ... all other command-level fields from old manifest
            Schema: nil,
        }},
    }
    return ret
}
```

### 2.3 validManifest (load.go)

Checks performed:
- `manifest.Name` must be non-empty
- For each command:
  - `command_name` must be non-empty
  - `files` must have at least one entry
  - Each file must have non-empty `os`, `arch`, and `path`
  - File paths sanitized via `util.ResolvePath()` (rejects path traversal)
  - `os` and `arch` are lowercased
  - `help` must be non-empty
  - If `schema` is present, `name` must be a valid schema type

---

## 3. Argument System

### 3.1 Valid Argument Types

Checked in `checkExtensionArgs` (load.go):

| Manifest Type | Aliases | Binary Packing | BOF Read Function | CNA bof_pack Char |
|---------------|---------|----------------|-------------------|-------------------|
| `"string"` | — | uint32 length + UTF-8 bytes + null | `BeaconDataExtract` | `z` |
| `"wstring"` | — | uint32 length + UTF-16LE bytes + null | (wide extract) | `Z` |
| `"integer"` | `"int"` | uint32 little-endian | `BeaconDataInt` | `i` |
| `"short"` | — | uint16 little-endian | `BeaconDataShort` | `s` |
| `"file"` | — | uint32 length + raw file bytes | `BeaconDataExtract` | `b` |

Both `"int"` and `"integer"` are accepted. They are functionally identical.

### 3.2 BOF Argument Parsing (argparser.go)

When the extension file has a `.o` extension (BOF), arguments are parsed as
**flags** using Go's `flag` package:

1. A `flag.FlagSet` is created with one flag per manifest argument
2. User provides args as `-- --name value` pairs
3. Each argument is processed by a type-specific handler:
   - `bofProcessStringArg` → `argsBuffer.AddString()`
   - `bofProcessWStringArg` → `argsBuffer.AddWString()`
   - `bofProcessIntArg` → `argsBuffer.AddInt(uint32)`
   - `bofProcessShortArg` → `argsBuffer.AddShort(uint16)`
   - `bofProcessFileArg` → reads file from disk → `argsBuffer.AddData()`
4. `bofFlagWasProvided` checks if a flag was explicitly set by the user
5. For string/wstring/file: value must be non-empty AND flag must have been visited

**CRITICAL: Argument order in the manifest `arguments` array determines the order
arguments are packed into the BOFArgsBuffer. The BOF reads them sequentially.
Wrong order = BOF reads garbage = CRASH.**

### 3.3 DLL Argument Parsing (load.go)

When the extension is a DLL (not `.o`), arguments are trivially handled:
```go
extensionArgs = []byte(strings.Join(args, " "))
```
No type-based parsing, no binary packing. Just space-joined string.

### 3.4 Default Value Handling (argparser.go — bofApplyDefaultValue)

When an argument is NOT provided by the user:

**If `optional == false`:** Error — argument name added to `missingRequiredArgs` list,
`ParseFlagArgumentsToBuffer` returns an error.

**If `optional == true`:** Apply default value:

If `argDef.Default` IS set in manifest:
- **string/wstring**: Tries string assertion, falls back to `fmt.Sprintf("%v", default)`.
  Writes via `AddString()` / `AddWString()`.
- **integer/int**: Tries float64 assertion first (JSON numbers unmarshal to float64
  via interface{}), then tries string parsing. Writes via `AddInt(uint32)`.
- **short**: Same as integer but uint16. Writes via `AddShort(uint16)`.
- **file**: Writes empty bytes `[]byte{}`. Default file path is meaningless.

If `argDef.Default` IS NOT set (nil):
- **string/wstring**: empty string `""`
- **integer/int**: `0`
- **short**: `0`
- **file**: empty `[]byte{}`

**IMPORTANT: Optional arguments are ALWAYS written to the buffer** (with their
default or zero value). They are not "skipped." The BOF always receives the full
argument list. This means the BOF's argument reading code must expect ALL arguments
to be present, regardless of optionality.

### 3.5 Optional Parameter Behavior

The `optional` field controls:

1. **Requirement enforcement**: Non-optional missing args → error returned to user
2. **Help text**: Optional args shown in `[BRACKETS]`, required in plain `UPPERCASE`
3. **Tab completion**: Optional args get `(optional)` suffix
4. **Buffer packing**: Optional args still written with default/zero value (see above)

### 3.6 Choices Field

`choices` provides tab-completion suggestions only. **Not enforced at runtime.**
When non-empty, the autocompleter offers those specific values instead of
generic type-based completion.

---

## 4. BOFArgsBuffer Binary Format (core/bof.go)

```go
type BOFArgsBuffer struct {
    Buffer *bytes.Buffer
}
```

Methods:
- `AddData([]byte)` — uint32 length prefix + raw bytes
- `AddShort(uint16)` — 2 bytes little-endian
- `AddInt(uint32)` — 4 bytes little-endian
- `AddString(string)` — uint32 length (including null) + UTF-8 bytes + 0x00
- `AddWString(string)` — encode to UTF-16LE, then uint32 length (including null pair) + UTF-16LE bytes + 0x0000
- `GetBuffer()` — prepends total buffer length as uint32, returns full byte slice

All integer writes use `binary.LittleEndian`.

---

## 5. Execution Flow

### 5.1 runExtensionCmd (load.go)

1. Get active session/beacon, determine OS/arch
2. Look up `ExtCommand` from loaded extensions map
3. Call `loadExtension` to ensure binary is loaded on implant:
   - Compute SHA256 of binary file
   - Check if already loaded (sessions only, not beacons)
   - If `DependsOn` is set, load the dependency (COFF loader) instead
   - Send `RegisterExtensionReq` RPC
4. Determine if BOF (`.o`) or DLL
5. **BOF path**: `getBOFArgs` creates BOFArgsBuffer, adds entrypoint string,
   adds BOF binary data, calls `ParseFlagArgumentsToBuffer` for user args
6. **DLL path**: joins args with spaces, uses command's own entrypoint
7. Send `CallExtensionReq` RPC with extension name (SHA256 hash), export, and packed args

### 5.2 Extension Identity on Implant

Extensions are registered on the implant by the **SHA256 hash of their binary file**,
not by name. This prevents loading duplicate copies of the same binary. Multiple
commands can share the same binary (e.g., multi-command manifest with different entrypoints).

---

## 6. Installation and Naming

### 6.1 Install Directory (install.go)

```go
packageID := manifestF.Name
if manifestF.PackageName != "" {
    packageID = manifestF.PackageName
}
installPath := filepath.Join(assets.GetExtensionsDir(), filepath.Base(packageID))
```

Priority: `package_name` > `name`.

### 6.2 Runtime File Resolution (load.go — getFileForTarget)

```go
if e.Manifest.RootPath != "" {
    filePath = path.Join(e.Manifest.RootPath, extFile.Path)
} else {
    filePath = path.Join(assets.GetExtensionsDir(), e.Manifest.Name, extFile.Path)
}
```

When `RootPath` is set (ALWAYS during normal loading via `LoadExtensionManifest`),
files are resolved relative to `RootPath`. The `Name`-based fallback is only used
when `RootPath` is empty — an edge case that does not occur in normal operation.

**Conclusion: `package_name != name` is safe in practice** because `RootPath` is
always set. But be aware of the fallback path.

### 6.3 Armory Tarball Naming (armory/parsers.go)

The armory constructs download URLs as:
```
<release_url>/releases/download/<tag>/<CommandName>.tar.gz
<release_url>/releases/download/<tag>/<CommandName>.minisig
```

Where `CommandName` comes from the **armory index** (`armory.json`), not from
the extension.json. However, the build scripts (make_bof.sh) typically use
`command_name` from extension.json to name the tarball, so they must stay in sync.

### 6.4 Extension Discovery (assets/extensions.go)

Scans `~/.sliver-client/extensions/`, finds any subdirectory containing an
`extension.json`, returns those paths. **Directory name does not matter for discovery.**

### 6.5 "Already Installed" Check (install.go)

Collects `command_name` values from all installed manifests. Uses `command_name`
for matching whether a package is already installed.

---

## 7. Command Registration (load.go — ExtensionRegisterCommand)

1. Validates argument types via `checkExtensionArgs`
2. Builds usage string with argument placeholders:
   - Optional args: `[ARG_NAME]`
   - Required args: `ARG_NAME`
3. Builds long help with argument descriptions
4. Creates a `cobra.Command` with `runExtensionCmd` as handler
5. Adds `--save` and `--timeout` flags automatically
6. Sets `ParseErrorsWhitelist.UnknownFlags = true` (allows `--` flag-style args to pass through)
7. Builds tab completers for both positional and dash-style arguments
8. Registers platform filter annotations from `Files` entries

---

## 8. CNA Cross-Reference Guide

### 8.1 bof_pack Format Characters

| Char | Type | Equivalent Manifest Type |
|------|------|-------------------------|
| `z` | null-terminated string | `"string"` |
| `Z` | null-terminated wide string (UTF-16LE) | `"wstring"` |
| `i` | 32-bit integer | `"integer"` or `"int"` |
| `s` | 16-bit short | `"short"` |
| `b` | binary blob (file contents) | `"file"` |

### 8.2 Mapping CNA to Manifest

1. Find the `bof_pack` call in the CNA command handler
2. The format string gives you type sequence (e.g., `"Zzis"` = wstring, string, int, short)
3. The arguments to `bof_pack` after the format string give you the values in order
4. **This order MUST match the `arguments` array order in extension.json**
5. The CNA may reorder user-supplied arguments before calling bof_pack — trace carefully

### 8.3 Common CNA Patterns

- `$1`, `$2`, etc. = positional user arguments
- `iff($N eq "", default, $N)` = optional argument with default
- `openFile(...)` = file picker dialog; maps to `"file"` type in manifest
- `readbof(...)` = helper that loads the correct arch .o file
- `bof_pack(...)` = packs arguments into binary format
- `beacon_inline_execute(...)` = executes the BOF with packed args

---

## 9. Safety Rules — Crash Scenarios

These manifest errors WILL crash the implant process:

### 9.1 Wrong Argument Order
If the manifest `arguments` array is in a different order than the BOF reads them
(via `BeaconDataExtract`/`BeaconDataInt`/`BeaconDataShort`), the BOF reads the
wrong bytes. A string read where an int is expected (or vice versa) produces garbage
data. Depending on how the BOF uses that data, this causes:
- Access violations (null/garbage pointers)
- Buffer overreads
- Immediate crash

### 9.2 Wrong Argument Type
If the manifest says `"string"` but the BOF calls `BeaconDataInt`, or vice versa:
- String packing writes length-prefix + bytes + null
- Int packing writes exactly 4 bytes
- The reader consumes the wrong number of bytes
- All subsequent argument reads are offset → cascading garbage → crash

### 9.3 Missing Arguments (Not Marked Optional)
If a required argument is missing, Sliver returns an error to the user. This is SAFE —
the BOF never executes. The danger is marking an argument as `optional` when the BOF
actually requires it: the BOF receives a zero/empty default and may crash on null pointer
dereference or division by zero.

### 9.4 Extra Arguments in Manifest
If the manifest defines MORE arguments than the BOF reads, the extra data sits in the
buffer unused. This is generally SAFE but wasteful. The BOF stops reading when it has
what it needs.

### 9.5 Fewer Arguments in Manifest
If the manifest defines FEWER arguments than the BOF expects, the BOF tries to read
past the end of the buffer. This causes a buffer underread → garbage or crash.

### 9.6 Wrong Entrypoint
If the entrypoint name doesn't match a function exported by the BOF, the COFF loader
cannot resolve it. Behavior depends on loader implementation — may error cleanly or crash.

### 9.7 Summary: When to STOP

**Do not produce a manifest if ANY of these are true:**
- You cannot determine the argument count with certainty from entry.c
- You cannot determine the argument order from entry.c
- You cannot determine argument types from entry.c (ambiguous BeaconData calls)
- The CNA and entry.c disagree on argument count/order/types and you cannot resolve which is correct
- The BOF uses conditional/variadic argument parsing that the manifest format cannot represent
- You are guessing about whether an argument is truly optional

**Flag for human review instead of guessing.**
