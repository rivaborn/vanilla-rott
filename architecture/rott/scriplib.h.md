# rott/scriplib.h

## File Purpose
Public header for the script parsing/tokenization subsystem. Declares the interface for loading script files and extracting tokens, supporting line-based text processing with lookahead capability.

## Core Responsibilities
- Define global state for active script buffers and parsing position
- Declare token extraction functions (sequential and with lookahead)
- Track parsing context (line number, end-of-script state)
- Support token availability checking

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| token | char[MAXTOKEN] | global | Current extracted token |
| name | char[MAXTOKEN*2] | global | Name buffer (double-sized token) |
| scriptbuffer | char* | global | Start of loaded script file in memory |
| script_p | char* | global | Current read position in script |
| scriptend_p | char* | global | End boundary of script buffer |
| scriptline | int | global | Current line number for error reporting |
| endofscript | boolean | global | Flag: reached end of file |
| tokenready | boolean | global | Flag: lookahead token pending from UnGetToken |

## Key Functions / Methods

### LoadScriptFile
- Signature: `void LoadScriptFile(char *filename)`
- Purpose: Load a script file from disk into memory
- Inputs: filename (path string)
- Outputs/Return: None (state mutation via globals)
- Side effects: Allocates memory, initializes scriptbuffer/script_p/scriptend_p globals, resets scriptline
- Calls: (not visible in header)
- Notes: Likely called once per script load phase

### GetToken
- Signature: `void GetToken(boolean crossline)`
- Purpose: Extract next token from script buffer
- Inputs: crossline (if true, token may span multiple lines; if false, stops at newline)
- Outputs/Return: None (result stored in global `token`)
- Side effects: Advances script_p, updates scriptline if crossline enabled, may set endofscript
- Calls: (not visible in header)
- Notes: Core parsing primitive

### GetTokenEOL
- Signature: `void GetTokenEOL(boolean crossline)`
- Purpose: Extract token and verify it terminates a logical line
- Inputs: crossline (line continuation behavior)
- Outputs/Return: None
- Side effects: Same as GetToken, possibly asserts/errors if EOL not found
- Calls: (not visible in header)
- Notes: Stricter variant for enforcing line-end semantics

### UnGetToken
- Signature: `void UnGetToken(void)`
- Purpose: Push back the last extracted token for re-reading (1-token lookahead)
- Inputs: None
- Outputs/Return: None
- Side effects: Sets tokenready flag; next GetToken call returns the same token
- Calls: (not visible in header)
- Notes: tokenready guards against double-lookahead

### TokenAvailable
- Signature: `boolean TokenAvailable(void)`
- Purpose: Check if a token can be read from current position
- Inputs: None
- Outputs/Return: Boolean (true if non-EOF)
- Side effects: None
- Calls: (not visible in header)
- Notes: Likely queries endofscript or peeks next character

## Control Flow Notes
Follows a classic single-pass lexer pattern: LoadScriptFile initializes global state once, then repeated GetToken calls drive parsing. UnGetToken enables 1-token lookahead for recursive-descent or backtracking parsers. scriptline enables error diagnostics tied to source location.

## External Dependencies
- None visible; header is self-contained declaration only.
