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
| `server/rpc/rpc-extensions.go` | Server-side capability gate for Reflektor (`CapabilityBOFV1` check) |
| `implant/sliver/handlers/handlers_bof.go` | Implant-side Reflektor BOF executor (cross-platform) |

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
    BOFExecutor     string               `json:"bof_executor"`
    Init            string               `json:"init"`
    RootPath        string               `json:"-"`
}
```

Everything is top-level. One manifest = one command. The `command_name`, `help`,
`files`, `arguments`, `entrypoint`, `depends_on`, `bof_executor`, and `init` all
sit at the top level alongside metadata.

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
    BOFExecutor string                 `json:"bof_executor"`
    Init        string                 `json:"init"`
    Schema      *packages.OutputSchema `json:"schema"`
    Manifest    *ExtensionManifest     // back-reference, not serialized
}
```

Metadata at top level, command(s) in `"commands"` array. New fields:
- `PackageName` — used for install directory naming
- `BOFExecutor` — selects the BOF execution engine (see Section 5)
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
  - `bof_executor` validation:
    - `""` (empty) and `"reflektor"` — accepted unconditionally
    - `"coff-loader"` — accepted only if `depends_on` is non-empty (otherwise rejected)
    - Any other value — rejected
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

### 5.1 BOF Executor and Execution Modes

Sliver supports three execution modes for extensions, determined by the
`planExtensionExecution` function in load.go:

| Mode | When | How |
|------|------|-----|
| `extensionExecutionNative` | Non-BOF (DLL/shared lib) | Direct DLL call via `CallExtensionReq` |
| `extensionExecutionCOFFLoader` | BOF with external COFF loader | Legacy path: register COFF loader, send BOF data in args |
| `extensionExecutionReflektor` | BOF with built-in Reflektor | Built-in implant executor: BOF data sent in `CallExtensionReq.BOFData` |

#### 5.1.1 planExtensionExecution Routing

A command is treated as a BOF if `bof_executor` is non-empty OR the file
extension is `.o`. The routing logic:

| `bof_executor` | `depends_on` | Implant has `CapabilityBOFV1` | Result |
|---|---|---|---|
| `""` (empty) | `"coff-loader"` | (doesn't matter) | **COFFLoader** — legacy route preserved |
| `""` (empty) | `""` (empty) | yes | **Reflektor** |
| `""` (empty) | `""` (empty) | no | **ERROR** — no execution path |
| `"coff-loader"` | non-empty | (doesn't matter) | **COFFLoader** |
| `"coff-loader"` | `""` (empty) | (doesn't matter) | **ERROR** — rejected at validation |
| `"reflektor"` | non-empty | yes | **Reflektor** (preferred) |
| `"reflektor"` | non-empty | no | **COFFLoader** — fallback |
| `"reflektor"` | `""` (empty) | yes | **Reflektor** |
| `"reflektor"` | `""` (empty) | no | **ERROR** — no fallback |

**Key rule**: Existing manifests with `"depends_on": "coff-loader"` and no
`bof_executor` field **always** use the COFF loader path, even if the implant
supports Reflektor. The source comment says: *"Existing manifests infer the
loader from depends_on. Keep that route even when the target supports the
built-in executor."*

**New footgun**: A `.o` file with neither `depends_on` nor `bof_executor` will
ONLY work on implants with `CapabilityBOFV1` (Reflektor). On older implants,
it fails with no fallback.

#### 5.1.2 Safe Manifest Configurations

From safest to most restrictive:

1. **`"depends_on": "coff-loader"` only** — Always uses COFF loader. Works on
   all implant versions. Windows-only. This is the traditional pattern.

2. **`"bof_executor": "reflektor"` + `"depends_on": "coff-loader"`** — Prefers
   Reflektor on capable implants, falls back to COFF loader on older ones.
   Cross-platform on new implants, Windows-only fallback. Best compatibility.

3. **`"bof_executor": "reflektor"` only** — Reflektor only. Cross-platform
   but fails on implants without `CapabilityBOFV1`. No fallback.

### 5.2 runExtensionCmd (load.go)

1. Get active session/beacon, determine OS/arch
2. Look up `ExtCommand` from loaded extensions map
3. Call `planExtensionExecution` to determine execution mode
4. If mode is NOT Reflektor, call `loadExtension` to ensure binary is loaded:
   - Compute SHA256 of binary file
   - Check if already loaded (sessions only, not beacons)
   - If `DependsOn` is set, load the dependency (COFF loader) instead
   - Send `RegisterExtensionReq` RPC
5. Route by mode:
   - **COFFLoader path**: `getBOFArgs` creates BOFArgsBuffer, adds entrypoint
     string, adds BOF binary data, calls `ParseFlagArgumentsToBuffer` for user
     args. Sends via `CallExtensionReq`.
   - **Reflektor path**: `ParseFlagArgumentsToBuffer` for user args only (no
     entrypoint/bindata wrapping). BOF binary sent directly in
     `CallExtensionReq.BOFData` and `IsBOF=true`. The server validates that the
     target implant has `CapabilityBOFV1` before forwarding.
   - **Native path**: joins args with spaces, uses command's own entrypoint.
     Sends via `CallExtensionReq`.

### 5.3 Extension Identity on Implant

Extensions are registered on the implant by the **SHA256 hash of their binary file**,
not by name. This prevents loading duplicate copies of the same binary. Multiple
commands can share the same binary (e.g., multi-command manifest with different entrypoints).

---

## 6. Directory Layout and File Paths

### 6.1 Flat Layout (Traditional)

The traditional pattern places `.o` files flat alongside `extension.json`:

```
bof-name/
├── extension.json        # "path": "name.x64.o"
├── name.x64.o
├── name.x86.o
└── LICENSE
```

Manifest `files` entries use bare filenames:
```json
"files": [
    {"os": "windows", "arch": "amd64", "path": "name.x64.o"},
    {"os": "windows", "arch": "386",   "path": "name.x86.o"}
]
```

Tarballs also use this flat structure. This remains the standard layout for
source-level manifests and Windows-only BOFs.

### 6.2 Sub-directory Layout (Cross-Platform)

The newer pattern uses `dist/{goos}/{goarch}/` sub-directories, primarily for
BOFs with cross-platform support:

```
bof-name/
├── extension.json        # "path": "dist/windows/amd64/name.o"
├── dist/
│   ├── windows/
│   │   ├── 386/name.o
│   │   ├── amd64/name.o
│   │   └── arm64/name.o
│   ├── linux/
│   │   ├── 386/name.o
│   │   ├── amd64/name.o
│   │   └── arm64/name.o
│   └── darwin/
│       ├── amd64/name.o
│       └── arm64/name.o
└── LICENSE
```

Manifest `files` entries use relative paths with directory separators:
```json
"files": [
    {"os": "windows", "arch": "amd64", "path": "dist/windows/amd64/name.o"},
    {"os": "linux",   "arch": "amd64", "path": "dist/linux/amd64/name.o"},
    {"os": "darwin",  "arch": "arm64", "path": "dist/darwin/arm64/name.o"}
]
```

Sliver's `install.go` explicitly supports this: `os.MkdirAll(filepath.Dir(dst))`
creates intermediate directories when extracting from tarballs. The comment
says: *"required for extensions with multiple dirs between the .o file and
the manifest."*

### 6.3 Platform Matrix

Traditional BOFs target Windows only (amd64 + 386). The expanded matrix adds:

| OS | Architectures | Notes |
|---|---|---|
| `windows` | `386`, `amd64`, `arm64` | arm64 is new; not all BOFs support it |
| `linux` | `386`, `amd64`, `arm64` | Requires Reflektor (no COFF loader on Linux) |
| `darwin` | `amd64`, `arm64` | Requires Reflektor (no COFF loader on macOS) |

Cross-platform targets only apply when the BOF has portable implementations.
Windows-only BOFs should stick to the traditional Windows entries.

**Build pipeline note**: The CS-Situational-Awareness-BOF packaging system
(`armory_packages.py`) rewrites `files` arrays and injects cross-platform
targets at build time. Source-level manifests in that repo still use flat
Windows-only paths. Other standalone BOF repos manage their own `files` entries.

### 6.4 File Path Resolution (load.go — getFileForTarget)

```go
if e.Manifest.RootPath != "" {
    filePath = path.Join(e.Manifest.RootPath, extFile.Path)
} else {
    filePath = path.Join(assets.GetExtensionsDir(), e.Manifest.Name, extFile.Path)
}
```

Both flat paths (`name.x64.o`) and sub-directory paths (`dist/windows/amd64/name.o`)
resolve correctly via `path.Join`. `RootPath` is always set during normal loading.

**Note on `package_name` vs `name`**: Because `RootPath` is always set during
normal loading, `package_name != name` is safe in practice — file resolution
uses `RootPath`, not `Name`. The `Name`-based fallback path only triggers when
`RootPath` is empty, which does not occur in normal operation.

---

## 7. Installation and Naming

### 7.1 Install Directory (install.go)

```go
packageID := manifestF.Name
if manifestF.PackageName != "" {
    packageID = manifestF.PackageName
}
installPath := filepath.Join(assets.GetExtensionsDir(), filepath.Base(packageID))
```

Priority: `package_name` > `name`.

### 7.2 Armory Tarball Naming (armory/parsers.go)

The armory constructs download URLs as:
```
<release_url>/releases/download/<tag>/<CommandName>.tar.gz
<release_url>/releases/download/<tag>/<CommandName>.minisig
```

Where `CommandName` comes from the **armory index** (`armory.json`), not from
the extension.json. However, the build scripts (make_bof.sh) typically use
`command_name` from extension.json to name the tarball, so they must stay in sync.

### 7.3 Extension Discovery (assets/extensions.go)

Scans `~/.sliver-client/extensions/`, finds any subdirectory containing an
`extension.json`, returns those paths. **Directory name does not matter for discovery.**

### 7.4 "Already Installed" Check (install.go)

Collects `command_name` values from all installed manifests. Uses `command_name`
for matching whether a package is already installed.

---

## 8. Command Registration (load.go — ExtensionRegisterCommand)

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

## 9. CNA Cross-Reference Guide

### 9.1 bof_pack Format Characters

| Char | Type | Equivalent Manifest Type |
|------|------|-------------------------|
| `z` | null-terminated string | `"string"` |
| `Z` | null-terminated wide string (UTF-16LE) | `"wstring"` |
| `i` | 32-bit integer | `"integer"` or `"int"` |
| `s` | 16-bit short | `"short"` |
| `b` | binary blob (file contents) | `"file"` |

### 9.2 Mapping CNA to Manifest

1. Find the `bof_pack` call in the CNA command handler
2. The format string gives you type sequence (e.g., `"Zzis"` = wstring, string, int, short)
3. The arguments to `bof_pack` after the format string give you the values in order
4. **This order MUST match the `arguments` array order in extension.json**
5. The CNA may reorder user-supplied arguments before calling bof_pack — trace carefully

### 9.3 Common CNA Patterns

- `$1`, `$2`, etc. = positional user arguments
- `iff($N eq "", default, $N)` = optional argument with default
- `openFile(...)` = file picker dialog; maps to `"file"` type in manifest
- `readbof(...)` = helper that loads the correct arch .o file
- `bof_pack(...)` = packs arguments into binary format
- `beacon_inline_execute(...)` = executes the BOF with packed args

### 9.4 Adaptix .axs Patterns

Adaptix C2 uses `.axs` scripts instead of CNA. Functionally equivalent for
manifest work — same bof_pack concept, different syntax:

- `ax.bof_pack("cstr,wstr,int", [arg1, arg2, arg3])` — comma-separated format
  string. Types: `cstr` (string), `wstr` (wstring), `int` (integer),
  `short` (short), `bytes` (file/binary data)
- `ax.create_command("name", "help text", "usage example")` — command definition
- `cmd.addArgString("name", true/false, "desc")` — string argument (true=required)
- `cmd.addArgInt("name", true/false, "desc")` — integer argument
- `cmd.addArgBool("--flag", "desc")` — boolean flag (packed as int: 0 or 1)
- `cmd.addArgFlagString("-f", "name", "desc", "default")` — optional flag with default
- `cmd.addArgFlagFile("--file", "name", required, "desc")` — file argument
- `parsed_json["name"]` — how the hook reads parsed args before packing
- `ax.execute_alias(id, cmdline, ...)` — executes the BOF

When both .axs and .cna exist, treat them the same: useful for help text,
optionality, and defaults, but entry.c is still the source of truth for
argument types, order, and count.

### 9.5 Passthrough Argument Pattern

Some BOFs (notably from Adaptix/Rubeus-style ports) take a single string via
`BeaconDataExtract` containing all parameters as `/key:value` pairs. The BOF
parses this string internally using its own helpers (e.g., `GetStrParam`,
`IsSetParam`).

Identifying the pattern:
- entry.c has exactly one `BeaconDataExtract` call after `BeaconDataParse`
- The extracted string is passed to an internal parser, not used directly
- The .axs/.cna uses `bof_pack("cstr", [params])` with a single concatenated string

Manifest implications:
- One `"string"` argument named `params` (or similar)
- The framework cannot validate, tab-complete, or provide help for individual
  `/key:value` parameters — that's all opaque to the manifest
- Optionality depends on whether the BOF handles an empty string gracefully
- Lower manifest risk (only one arg to get right), higher operator risk
  (no framework-level input validation)

Do NOT attempt to break passthrough args into discrete manifest arguments
unless you're also modifying the BOF's C source to use discrete
`BeaconDataExtract` calls. The manifest argument order must match what the
BOF actually reads.

---

## 10. Safety Rules — Crash Scenarios

These manifest errors WILL crash the implant process:

### 10.1 Wrong Argument Order
If the manifest `arguments` array is in a different order than the BOF reads them
(via `BeaconDataExtract`/`BeaconDataInt`/`BeaconDataShort`), the BOF reads the
wrong bytes. A string read where an int is expected (or vice versa) produces garbage
data. Depending on how the BOF uses that data, this causes:
- Access violations (null/garbage pointers)
- Buffer overreads
- Immediate crash

### 10.2 Wrong Argument Type
If the manifest says `"string"` but the BOF calls `BeaconDataInt`, or vice versa:
- String packing writes length-prefix + bytes + null
- Int packing writes exactly 4 bytes
- The reader consumes the wrong number of bytes
- All subsequent argument reads are offset → cascading garbage → crash

### 10.3 Missing Arguments (Not Marked Optional)
If a required argument is missing, Sliver returns an error to the user. This is SAFE —
the BOF never executes. The danger is marking an argument as `optional` when the BOF
actually requires it: the BOF receives a zero/empty default and may crash on null pointer
dereference or division by zero.

### 10.4 Extra Arguments in Manifest
If the manifest defines MORE arguments than the BOF reads, the extra data sits in the
buffer unused. This is generally SAFE but wasteful. The BOF stops reading when it has
what it needs.

### 10.5 Fewer Arguments in Manifest
If the manifest defines FEWER arguments than the BOF expects, the BOF tries to read
past the end of the buffer. This causes a buffer underread → garbage or crash.

### 10.6 Wrong Entrypoint
If the entrypoint name doesn't match a function exported by the BOF, the loader
(COFF loader or Reflektor) cannot resolve it. Behavior depends on loader
implementation — may error cleanly or crash.

### 10.7 Compiler-Induced Failures

A BOF can pass all manifest-level validation (correct COFF format, correct
machine type, `go` entry symbol present) but still crash at runtime due to
compiler-specific issues:

- **LLVM/Clang-based cross-compilers** (including OLLVM) may produce object
  files with subtly broken symbol resolution for common libc functions
  (e.g., `memcpy`, `malloc`). The `.o` file looks correct to `nm` and `file`
  but crashes when the COFF loader resolves imports at runtime.
- **GCC version differences**: Newer GCC versions (16+) promote some warnings
  to errors (e.g., `-Wint-conversion`, `-Wincompatible-function-pointer-types`).
  BOFs written for older compilers may need additional `-Wno-*` flags.

When diagnosing BOFs that pass verification but fail at runtime, consider the
compiler before assuming manifest errors. Test with the same compiler the
upstream project uses (usually `x86_64-w64-mingw32-gcc`).

### 10.8 Summary: When to STOP

**Do not produce a manifest if ANY of these are true:**
- You cannot determine the argument count with certainty from entry.c
- You cannot determine the argument order from entry.c
- You cannot determine argument types from entry.c (ambiguous BeaconData calls)
- The CNA and entry.c disagree on argument count/order/types and you cannot resolve which is correct
- The BOF uses conditional/variadic argument parsing that the manifest format cannot represent
- You are guessing about whether an argument is truly optional

**Flag for human review instead of guessing.**
