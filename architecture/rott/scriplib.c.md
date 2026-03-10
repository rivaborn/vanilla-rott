# rott/scriplib.c

## File Purpose
Script file parser and token extraction library. Loads script files into memory and provides functions to tokenize and iterate through script content, with support for line tracking, lookahead, and comment handling for a configuration/script processing system.

## Core Responsibilities
- Load entire script files from disk into managed memory buffer
- Tokenize script content by extracting whitespace-delimited words
- Track current line number for error reporting
- Skip whitespace and semicolon-prefixed comment lines
- Provide lookahead capability via token pushback (UnGetToken)
- Support both single-token and end-of-line (full line) extraction modes
- Detect end-of-script conditions

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| token | char[MAXTOKEN] | global | Current extracted token buffer |
| name | char[MAXTOKEN\*2] | global | Larger buffer for line-based reads (GetTokenEOL) |
| scriptfilename | char[30] | global | Name of loaded script file |
| scriptbuffer | char\* | global | Entire script file loaded into memory |
| script_p | char\* | global | Current read position in script |
| scriptend_p | char\* | global | End boundary pointer of script buffer |
| scriptline | int | global | Current line number in script |
| endofscript | boolean | global | Flag: reached end of file |
| tokenready | boolean | global | Flag: token from UnGetToken is pending |

## Key Functions / Methods

### LoadScriptFile
- **Signature**: `void LoadScriptFile(char *filename)`
- **Purpose**: Load an entire script file into memory and initialize parsing state
- **Inputs**: `filename` — path to script file to load
- **Outputs/Return**: None; modifies global state
- **Side effects**: 
  - Calls `LoadFile()` to read file into `scriptbuffer`; sets `script_p`, `scriptend_p` to buffer boundaries
  - Copies filename to `scriptfilename[30]`
  - Resets `scriptline` to 1, `endofscript` to false, `tokenready` to false
- **Calls**: `LoadFile()`, `strcpy()`
- **Notes**: Allocates memory via `LoadFile()`; caller responsible for freeing buffer. No validation that file exists or is readable.

### GetToken
- **Signature**: `void GetToken(boolean crossline)`
- **Purpose**: Extract next whitespace-delimited token from script buffer
- **Inputs**: `crossline` — if false, error if token spans line boundary; if true, allow newlines within skipped whitespace
- **Outputs/Return**: Token written to global `token[MAXTOKEN]` buffer; `\0`-terminated
- **Side effects**:
  - Advances `script_p` through buffer
  - Increments `scriptline` on each `\n` encountered
  - Sets `endofscript = true` when buffer exhausted
  - Clears `tokenready = false`
  - Calls `Error()` on line boundary violation (if `!crossline`) or token overflow
- **Calls**: `Error()`
- **Notes**:
  - If `tokenready == true` (from prior UnGetToken), returns immediately without parsing
  - Skips all whitespace (chars ≤ 32) and comment lines (lines starting with `;`)
  - Token spans from first char > 32 to next char ≤ 32 or `;` (exclusive)
  - Errors if token exceeds MAXTOKEN bytes

### GetTokenEOL
- **Signature**: `void GetTokenEOL(boolean crossline)`
- **Purpose**: Extract entire remainder of current line as a single token, preserving internal whitespace
- **Inputs**: `crossline` — same as GetToken
- **Outputs/Return**: Token written to global `name[MAXTOKEN*2]` buffer; `\0`-terminated
- **Side effects**: Same as GetToken (advances `script_p`, increments `scriptline`, sets `endofscript`, clears `tokenready`)
- **Calls**: `Error()`
- **Notes**:
  - Reads characters while `*script_p >= 32` (stops at newline or control char)
  - Unlike GetToken, includes internal whitespace; only breaks on newline
  - Uses larger `name[]` buffer (256 bytes vs. 128 for `token[]`)
  - Errors if token exceeds MAXTOKEN*2 bytes

### UnGetToken
- **Signature**: `void UnGetToken(void)`
- **Purpose**: Mark current token as unconsumed; return it on next GetToken call
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Sets `tokenready = true`
- **Calls**: None
- **Notes**: Simple flag setter enabling lookahead. Parser can call GetToken, then UnGetToken to "undo" the read.

### TokenAvailable
- **Signature**: `boolean TokenAvailable(void)`
- **Purpose**: Check if another token exists on current line without consuming it
- **Inputs**: None
- **Outputs/Return**: true if non-whitespace/non-comment token remains on line; false if EOL or EOF
- **Side effects**: None (read-only lookahead; does not modify `script_p`)
- **Calls**: None
- **Notes**:
  - Returns false if next non-whitespace char is `;` (comment), `\n` (EOL), or reached `scriptend_p`
  - Used by higher-level parser to conditionally consume tokens

## Control Flow Notes
Fits into **initialization/loading phase**: script file is loaded once (LoadScriptFile), then parsed via repeated GetToken calls, likely processing game configuration, map definitions, or script definitions. The line-tracking and comment support suggest this parses human-readable configuration files with comments. The lookahead (UnGetToken, TokenAvailable) indicates a simple recursive-descent or LL(1) parser.

## External Dependencies
- **Notable includes**:
  - `rt_def.h` — defines `boolean`, `byte`, game constants
  - `scriplib.h` — declares these five functions; defines MAXTOKEN
  - `rt_util.h` — declares `Error()` (printf-like error reporting)
  - `memcheck.h` — memory debugging wrapper (optional)
  - Platform-specific: `<io.h>`, `<dos.h>`, `<fcntl.h>` for DOS file I/O; `<libc.h>` for NeXTSTEP UNIX

- **Defined elsewhere**:
  - `LoadFile(filename, **bufferptr)` — file I/O utility; returns file size
  - `Error(format, ...)` — error reporting; likely aborts on error
  - `strcpy()` — C standard library
