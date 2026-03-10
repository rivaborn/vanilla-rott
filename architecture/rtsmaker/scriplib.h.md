# rtsmaker/scriplib.h

## File Purpose
Header file for a script parsing library used in the RTS tool. Provides a token-based lexer interface to load and parse script files from disk into memory, with functions to retrieve tokens sequentially and track parsing state.

## Core Responsibilities
- Load script files into memory buffers
- Tokenize script content by extracting tokens separated by whitespace/delimiters
- Manage parsing state (current position, script boundaries, line tracking)
- Provide token retrieval and ungetting (pushback) operations
- Track EOF and parsing completion status

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|--------------------------------|---------|
| `token` | `char[MAXTOKEN]` | global | Buffer holding the current extracted token |
| `scriptbuffer` | `char *` | global | Pointer to the entire loaded script in memory |
| `script_p` | `char *` | global | Current read position pointer within the script |
| `scriptend_p` | `char *` | global | End boundary pointer of the script buffer |
| `grabbed` | `int` | global | Flag indicating whether a token is buffered in `token[]` |
| `scriptline` | `int` | global | Current line number for error reporting |
| `endofscript` | `boolean` | global | Flag indicating EOF reached during parsing |

## Key Functions / Methods

### LoadScriptFile
- Signature: `void LoadScriptFile(char *filename)`
- Purpose: Load an entire script file from disk into memory and initialize parsing state
- Inputs: `filename` – path to the script file to load
- Outputs/Return: None (initializes global state)
- Side effects (global state, I/O, alloc): Reads file into `scriptbuffer`, sets `script_p`, `scriptend_p`, `scriptline`, `endofscript`; allocates memory
- Calls (direct calls visible in this file): Not visible (defined elsewhere)
- Notes: Must be called before `GetToken()` to initialize parsing state

### GetToken
- Signature: `void GetToken(boolean crossline)`
- Purpose: Extract the next token from the script and store it in the global `token[]` buffer
- Inputs: `crossline` – if true, allows token parsing to cross line boundaries; if false, stops at newlines
- Outputs/Return: None (populates global `token[]` and updates `script_p`, `scriptline`)
- Side effects (global state, I/O, alloc): Advances `script_p`, updates `scriptline` on newlines, may set `endofscript`
- Calls (direct calls visible in this file): Not visible (defined elsewhere)
- Notes: Assumes `LoadScriptFile()` called first; reads until whitespace or delimiter

### UnGetToken
- Signature: `void UnGetToken(void)`
- Purpose: "Push back" the current token so the next `GetToken()` call returns the same token again
- Inputs: None
- Outputs/Return: None
- Side effects (global state, I/O, alloc): Modifies `grabbed` flag; rewinds `script_p`
- Calls (direct calls visible in this file): Not visible (defined elsewhere)
- Notes: Useful for lookahead parsing; single-token pushback (likely no multi-token queue)

### TokenAvailable
- Signature: `boolean TokenAvailable(void)`
- Purpose: Check whether more tokens remain to be parsed
- Inputs: None
- Outputs/Return: `boolean` – true if tokens remain, false if EOF
- Side effects (global state, I/O, alloc): None (read-only check of `endofscript` state)
- Calls (direct calls visible in this file): Not visible (defined elsewhere)
- Notes: Used to drive parsing loops

## Control Flow Notes
This module is initialization/load-time infrastructure. Typical usage: `LoadScriptFile()` → loop `while(TokenAvailable())` → `GetToken()` → process token. `UnGetToken()` supports backtracking in recursive descent or predictive parsing.

## External Dependencies
- `cmdlib.h` – provides basic types (`byte`, `boolean`) and utility functions (`Error`, memory/file I/O)
- Global state and function implementations defined elsewhere (not in this header)
