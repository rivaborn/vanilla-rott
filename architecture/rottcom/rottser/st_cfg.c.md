# rottcom/rottser/st_cfg.c

## File Purpose
Loads and parses serial/modem configuration from SETUP.ROT and ROTT.ROT script files during game initialization. Extracts modem parameters (init/hangup strings, baud rate, COM port, IRQ, UART) and phone number, populating global variables used by the serial communication layer.

## Core Responsibilities
- Resolve configuration file paths using environment variables
- Load and parse script-based configuration files (SETUP.ROT, ROTT.ROT)
- Validate expected parameter tokens in configuration
- Extract modem initialization and hangup strings with fallback defaults
- Parse serial port hardware settings (baud rate, COM port, IRQ, UART address)
- Read phone number/dial string for modem connections
- Handle "not configured" markers (~) and provide sensible defaults

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| CONFIG | const char* | static | Filename "SETUP.ROT" for modem/serial config |
| ROTT | const char* | static | Filename "ROTT.ROT" for phone number config |
| ApogeePath | const char* | static | Environment variable name "APOGEECD" for base path |

## Key Functions / Methods

### GetPathFromEnvironment
- **Signature:** `void GetPathFromEnvironment( char *fullname, const char *envname, const char *filename )`
- **Purpose:** Constructs absolute file path by resolving environment variable and appending filename with backslash separator.
- **Inputs:** `envname` (env variable name), `filename` (file to append)
- **Outputs/Return:** Result written to `fullname` buffer
- **Side effects:** Calls `getenv()`, writes to caller's buffer
- **Calls:** `getenv()`, `strlen()`, `strcpy()`, `strcat()`
- **Notes:** Assumes sufficient buffer space; silently uses filename alone if environment variable unset

### CheckParameter
- **Signature:** `void CheckParameter (const char * s1, const char * file)`
- **Purpose:** Validates that current token matches expected parameter name; calls Error() if mismatch.
- **Inputs:** `s1` (expected token), `file` (filename for error message)
- **Outputs/Return:** None
- **Side effects:** Terminates execution via Error() on mismatch
- **Calls:** `strcmpi()`, `Error()`
- **Notes:** Used to verify parser is reading expected config tokens in order

### ReadSetup
- **Signature:** `void ReadSetup (void)`
- **Purpose:** Main initialization function that loads SETUP.ROT (modem settings) and ROTT.ROT (phone number) configuration files and populates global serial/modem state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Loads script files, sets globals (`initstring`, `hangupstring`, `dialstring`, `baudrate`, `comport`, `irq`, `uart`, `pulse`), frees `scriptbuffer`
- **Calls:** `GetPathFromEnvironment()`, `access()`, `LoadScriptFile()`, `GetToken()`, `GetTokenEOL()`, `CheckParameter()`, `atol()`, `atoi()`, `strcmpi()`, `sscanf()`, `Error()`, `free()`
- **Notes:** Tilde (~) marks unconfigured params; defaults to "ATZ" for init/hangup if modem disabled; expects strict token order; terminates if files not found

## Control Flow Notes
Called during game/server initialization to configure serial port and modem hardware. ReadSetup() synchronously loads two configuration files and populates module-level globals used by port.c and sermodem.c for hardware communication.

## External Dependencies
- **Local headers:** global.h, scriplib.h, port.h, sersetup.h, sermodem.h
- **Standard C / DOS:** stdio.h, string.h, stdlib.h, dos.h, io.h, fcntl.h, ctype.h, dir.h, process.h, errno.h, sys/stat.h
- **Defined elsewhere:** `Error()`, `LoadScriptFile()`, `GetToken()`, `GetTokenEOL()` (scriplib module), `access()` (libc), external globals `token[]`, `name[]`, `scriptbuffer` (scriplib), `initstring[]`, `dialstring[]`, `hangupstring[]`, `pulse` (sermodem), `comport`, `irq`, `uart`, `baudrate` (port), `usemodem` (sersetup)
