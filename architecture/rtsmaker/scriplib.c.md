# rtsmaker/scriplib.c

## File Purpose
Provides a tokenizing script parser for reading configuration/script files. Loads entire scripts into memory and extracts whitespace-delimited tokens with support for comments and line tracking.

## Core Responsibilities
- Load script files from disk into memory (`LoadScriptFile`)
- Extract and buffer individual tokens from script content
- Track parsing state (current position, line number, end-of-file)
- Skip whitespace, newlines, and semicolon-delimited comments
- Support token lookahead/pushback via `UnGetToken`
- Validate line completeness when `crossline=false`

## Key Types / Data Structures
None (uses basic C types only).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `token` | `char[MAXTOKEN]` | global | Current token buffer |
| `scriptbuffer` | `char*` | global | Pointer to entire loaded script file |
| `script_p` | `char*` | global | Current parse position in script |
| `scriptend_p` | `char*` | global | End boundary of script buffer |
| `scriptline` | `int` | global | Current line number (for error reporting) |
| `endofscript` | `boolean` | global | True when end of file reached |
| `tokenready` | `boolean` | global | True if `UnGetToken()` was called and token awaits reuse |
| `grabbed` | `int` | global | Declared but unused in this file |

## Key Functions / Methods

### LoadScriptFile
- **Signature:** `void LoadScriptFile (char *filename)`
- **Purpose:** Load an entire script file into memory and initialize parsing state
- **Inputs:** `filename` – path to script file
- **Outputs/Return:** None (results stored in global state)
- **Side effects:** Allocates memory via `LoadFile()`, updates `scriptbuffer`, `script_p`, `scriptend_p`, `scriptline`, `endofscript`, `tokenready`
- **Calls:** `LoadFile()` (from cmdlib)
- **Notes:** Initializes `scriptline=1`, `endofscript=false`, `tokenready=false`

### GetToken
- **Signature:** `void GetToken (boolean crossline)`
- **Purpose:** Extract next token from script, advancing parse position
- **Inputs:** `crossline` – if false, error on incomplete line; if true, allow tokens spanning line boundaries
- **Outputs/Return:** Result stored in global `token` buffer; sets `endofscript=true` on EOF
- **Side effects:** Updates `script_p`, `scriptline`, `tokenready`, `endofscript`
- **Calls:** `Error()` (from cmdlib, on parse errors)
- **Notes:** Skips whitespace (ASCII ≤32) and comments (lines starting with `;`). Returns early if `tokenready` is true (pushback state). Errors if token exceeds `MAXTOKEN` or line is incomplete without `crossline` flag.

### UnGetToken
- **Signature:** `void UnGetToken (void)`
- **Purpose:** Signal that the current token should not be consumed; next `GetToken()` returns it again
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `tokenready=true`
- **Calls:** None
- **Notes:** Must be called immediately after a `GetToken()`. Enables single-token lookahead pattern.

### TokenAvailable
- **Signature:** `boolean TokenAvailable (void)`
- **Purpose:** Check if another token remains on the current line (without advancing parse position)
- **Inputs:** None
- **Outputs/Return:** `true` if a non-whitespace, non-comment token precedes end-of-line or EOF
- **Side effects:** None (read-only scan via `search_p`)
- **Calls:** None
- **Notes:** Stops scanning at newline (`\n`), returning `false` immediately. Returns `false` if next non-whitespace char is `;` (comment).

## Control Flow Notes
This is a **parsing utility library** used at build/setup time (part of `rtsmaker`). It has no inherent control flow within a game loop; callers invoke `LoadScriptFile()` once, then repeatedly call `GetToken()`/`TokenAvailable()` to parse the loaded script sequentially.

## External Dependencies
- **Includes:** `cmdlib.h` (error handling, file I/O), platform-specific: `libc.h` (NeXT/Unix), `io.h`/`dos.h`/`fcntl.h` (DOS/Windows)
- **Defined elsewhere:** `Error()`, `LoadFile()` (from cmdlib)
- **Types from cmdlib.h:** `boolean`, `byte`
