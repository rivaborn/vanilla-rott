# rottcom/rottser/scriplib.h

## File Purpose
Public header for a script parsing library. Declares global variables and functions for loading and tokenizing script files in a token-stream model. This is a foundational utility for configuration or scripting systems within the engine.

## Core Responsibilities
- Expose global script parsing state (buffer pointers, current token, line tracking)
- Provide script file loading and initialization
- Declare token acquisition and lookahead functions
- Track EOF and token-availability state

## Key Types / Data Structures
None (this is a header declaring external state and functions; no new types defined).

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|------|---------|
| `token` | `char[MAXTOKEN]` | global | Stores the current parsed token string |
| `name` | `char[MAXTOKEN*2]` | global | Auxiliary name buffer (double capacity) |
| `scriptbuffer` | `char*` | global | Start of entire script buffer in memory |
| `script_p` | `char*` | global | Current read position in script |
| `scriptend_p` | `char*` | global | End boundary of script buffer |
| `scriptline` | `int` | global | Current line number for error reporting |
| `endofscript` | `boolean` | global | Flag indicating end-of-file reached |
| `tokenready` | `boolean` | global | Flag: true if `UnGetToken()` was just called |

## Key Functions / Methods

### LoadScriptFile
- Signature: `void LoadScriptFile(char *filename)`
- Purpose: Initialize script parsing by loading a file into the global script buffer
- Inputs: `filename` — path to script file
- Outputs/Return: None (state set in global variables)
- Side effects: Allocates/loads script buffer; sets `scriptbuffer`, `script_p`, `scriptend_p`, `scriptline`
- Calls: Not inferable from header
- Notes: Called once at initialization; assumes caller manages memory (likely via `SafeMalloc` from global.h)

### GetToken
- Signature: `void GetToken(boolean crossline)`
- Purpose: Advance to next token and store it in the global `token` buffer
- Inputs: `crossline` — whether tokenization can cross line boundaries
- Outputs/Return: None (result in global `token`)
- Side effects: Advances `script_p`, increments `scriptline`, sets `endofscript`
- Calls: Not inferable from header
- Notes: Called in main parse loop; `tokenready` implicitly reset to false

### GetTokenEOL
- Signature: `void GetTokenEOL(boolean crossline)`
- Purpose: Variant of `GetToken()` that enforces end-of-line after the token
- Inputs: `crossline` — line crossing mode
- Outputs/Return: None (result in global `token`)
- Side effects: Similar to `GetToken()`; may error if non-EOL content follows
- Calls: Not inferable from header
- Notes: Used when a token must be the last meaningful content on a line

### UnGetToken
- Signature: `void UnGetToken(void)`
- Purpose: Push back the current token for re-reading on next `GetToken()` call
- Inputs: None
- Outputs/Return: None
- Side effects: Sets `tokenready` to true; does not advance `script_p`
- Calls: Not inferable from header
- Notes: 1-token lookahead; cannot be called multiple times in succession

### TokenAvailable
- Signature: `boolean TokenAvailable(void)`
- Purpose: Check whether a token can be read without reaching EOF
- Inputs: None
- Outputs/Return: `boolean` — true if more tokens exist
- Side effects: None (read-only check)
- Calls: Not inferable from header
- Notes: Allows non-blocking lookahead before calling `GetToken()`

## Control Flow Notes
This module is a utility layer: **initialization** phase calls `LoadScriptFile()`; **parse** phase uses `GetToken()/GetTokenEOL()` in a loop until `endofscript` or error. The `UnGetToken()` / `TokenAvailable()` pair enables single-token lookahead for parsers needing to decide branch conditions. No render/shutdown involvement inferable.

## External Dependencies
- `#include "global.h"` — for `boolean` typedef and utility functions (`SafeMalloc`, file I/O)
- Symbols defined elsewhere: `boolean`, `Error()`, file I/O primitives
