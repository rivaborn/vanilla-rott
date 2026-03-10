# rottcom/rottser/scriplib.c

## File Purpose
Script file tokenizer and parser for the ROTT game engine. Provides utilities to load configuration/script files and extract tokens while tracking line numbers and managing parsing state.

## Core Responsibilities
- Load script files into memory and initialize parser state
- Tokenize input by extracting whitespace-delimited words
- Support both standard tokens and end-of-line tokens (rest of line)
- Skip comments (semicolon-delimited) and whitespace
- Track parse position, line numbers, and EOF state
- Implement one-token lookahead via UnGetToken

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `token` | `char[MAXTOKEN]` | static | Current extracted token buffer |
| `name` | `char[MAXTOKEN*2]` | static | Current EOL token buffer (larger for rest-of-line reads) |
| `scriptbuffer` | `char*` | static | Pointer to loaded script file content |
| `script_p` | `char*` | static | Current parse position within script |
| `scriptend_p` | `char*` | static | End boundary of script buffer |
| `scriptline` | `int` | static | Current line number (for error reporting) |
| `endofscript` | `boolean` | static | EOF flag |
| `tokenready` | `boolean` | static | Lookahead flag; true if UnGetToken was called |

## Key Functions / Methods

### LoadScriptFile
- **Signature:** `void LoadScriptFile (char *filename)`
- **Purpose:** Initialize script parsing by loading a file into memory and resetting parser state
- **Inputs:** `filename` – path to script file
- **Outputs/Return:** void (modifies global parse state)
- **Side effects:** Calls `LoadFile` to read file into `scriptbuffer`; resets `script_p`, `scriptend_p`, `scriptline=1`, `endofscript=false`, `tokenready=false`
- **Calls:** `LoadFile`
- **Notes:** No error handling for load failures; assumes `LoadFile` reports errors via `Error()` if needed

### GetToken
- **Signature:** `void GetToken (boolean crossline)`
- **Purpose:** Extract next whitespace-delimited token, skipping whitespace and inline comments
- **Inputs:** `crossline` – if false, errors if token crosses line boundary
- **Outputs/Return:** void (result in global `token` buffer, null-terminated)
- **Side effects:** Advances `script_p`; increments `scriptline` on newlines; sets `endofscript` on EOF; clears `tokenready`
- **Calls:** `Error` (if buffer overflow or line boundary violation)
- **Notes:** Treats newlines and characters ≤32 as delimiters; semicolon starts comment; uses `goto skipspace` for comment skip logic

### GetTokenEOL
- **Signature:** `void GetTokenEOL (boolean crossline)`
- **Purpose:** Extract rest of current line as a single token (preserving internal spaces)
- **Inputs:** `crossline` – if false, errors if token crosses line boundary
- **Outputs/Return:** void (result in global `name` buffer, null-terminated)
- **Side effects:** Advances `script_p` to EOL; increments `scriptline` on newlines; sets `endofscript` on EOF; clears `tokenready`
- **Calls:** `Error` (if buffer overflow or line boundary violation)
- **Notes:** Identical logic to `GetToken` except captures characters until EOL (no space delimiter); uses larger buffer (`MAXTOKEN*2`)

### UnGetToken
- **Signature:** `void UnGetToken (void)`
- **Purpose:** Signal that the current token should be re-read on the next `GetToken` call
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Sets `tokenready = true`
- **Calls:** none
- **Notes:** Simple one-token lookahead mechanism; `GetToken` returns immediately if `tokenready` is set

### TokenAvailable
- **Signature:** `boolean TokenAvailable (void)`
- **Purpose:** Check if another token exists on the current line without consuming it
- **Inputs:** none
- **Outputs/Return:** `boolean` – true if another token or non-comment content remains on line
- **Side effects:** none (read-only lookahead)
- **Calls:** none
- **Notes:** Returns false if EOL, EOF, or only comments remain; does not advance `script_p`

## Control Flow Notes
This is a utility library with no frame loop participation. Typical usage: `LoadScriptFile()` initializes parsing, then calling code repeatedly invokes `GetToken()` / `GetTokenEOL()` in a loop until `endofscript == true`. `UnGetToken()` and `TokenAvailable()` support conditional or peekahead logic.

## External Dependencies
- **`LoadFile`** (declared in `global.h`, defined elsewhere) – loads file into memory
- **`Error`** (declared in `global.h`, defined elsewhere) – reports parsing errors
- Platform headers: `io.h`, `dos.h`, `fcntl.h` (DOS/NeXT conditional includes)
