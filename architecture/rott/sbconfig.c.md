# rott/sbconfig.c

## File Purpose
Implements SpaceTec IMC Spaceware button configuration parsing and fixed-point arithmetic value warping. Reads button mappings and warp range definitions from a config file, stores them in static globals, and provides query/transformation functions to scale input values through piecewise-linear lookup tables.

## Core Responsibilities
- Parse button and warp range configuration from text file
- Store configuration in file-static globals (button names, warp records)
- Provide query functions to retrieve button names and warp ranges
- Implement compiler-specific fixed-point multiplication (16.16 format)
- Apply piecewise-linear warp transformations to short integer values
- Parse fixed-point and integer literals from config file strings

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `fixed` | typedef | 16.16 fixed-point number (long, from sbconfig.h) |
| `WarpRange` | struct | Single range segment: low/high input bounds, multiplier coefficient |
| `WarpRecord` | struct | Named warp configuration: name, pointer to WarpRange array, count |
| `SbButtonNames` | array | Static array of six predefined button name strings |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cfgFileVersion` | int | static | Config file format version (currently unused) |
| `cfgButtons` | char[][MAX_STRING_LENGTH] | static | Array of button label strings, indexed by button number |
| `pCfgWarps` | WarpRecord* | static | Pointer to dynamically allocated warp record array |
| `nCfgWarps` | int | static | Count of warp records loaded |

## Key Functions / Methods

### FIXED_MUL
- **Signature:** `fixed FIXED_MUL(fixed a, fixed b)` (three compiler-specific implementations: Borland ASM, MSC pure C, Watcom pragma)
- **Purpose:** Multiply two 16.16 fixed-point numbers, return 16.16 result.
- **Inputs:** Two fixed-point operands.
- **Outputs/Return:** Fixed-point product (both upper and lower 16 bits).
- **Side effects:** None.
- **Calls:** None (inline assembly or pure arithmetic).
- **Notes:** Critical for fixed-point arithmetic. Borland and Watcom use inline ASM (imul + shrd); MSC7 fallback uses manual bit decomposition to avoid 32-bit instruction limitations.

### StrToFx1616
- **Signature:** `static fixed StrToFx1616(char *string, char **ret_string)`
- **Purpose:** Parse a decimal number string (including fractional part) into 16.16 fixed-point.
- **Inputs:** String pointer, optional return string pointer.
- **Outputs/Return:** Fixed-point value; updates ret_string to point after parsed token.
- **Side effects:** None.
- **Calls:** None (character inspection loop).
- **Notes:** Supports [+-]?[0-9]+(.[0-9]*)? syntax. Handles sign, converts fraction part to fixed by scaling by 10^places and shifting.

### GetWarpLevels
- **Signature:** `static char *GetWarpLevels(char *string, WarpRange *pw)`
- **Purpose:** Parse a single warp range entry: `{low, high, multiplier}`.
- **Inputs:** String, optional WarpRange pointer to populate.
- **Outputs/Return:** Pointer to character after closing `}`, or NULL on parse error.
- **Side effects:** Writes to WarpRange if pw non-NULL.
- **Calls:** `strtol`, `StrToFx1616`, `isspace`.
- **Notes:** Expects exact format; allows whitespace around commas.

### GetWarp
- **Signature:** `static int GetWarp(char *string, WarpRecord *pRecord)`
- **Purpose:** Parse complete warp record: `{range1, range2, ...}` with dynamic reallocation of range array.
- **Inputs:** String, WarpRecord pointer to populate.
- **Outputs/Return:** 1 on success, 0 on error (early EOL, malloc/realloc failure).
- **Side effects:** Allocates/reallocates heap memory for WarpRange array; populates WarpRecord fields.
- **Calls:** `malloc`, `realloc`, `free`, `GetWarpLevels`, `isspace`.
- **Notes:** Grows WarpRange array incrementally; frees on error.

### SbConfigParse
- **Signature:** `int SbConfigParse(char *filename)`
- **Purpose:** Load and parse entire config file, populating cfgButtons and pCfgWarps globals.
- **Inputs:** Config filename (uses DEFAULT_CONFIG_FILENAME if NULL).
- **Outputs/Return:** 1 on success, 0 if file not found.
- **Side effects:** Updates cfgFileVersion, cfgButtons[], pCfgWarps, nCfgWarps; allocates heap.
- **Calls:** `fopen`, `fgets`, `strtok`, `stricmp`, `strncpy`, `GetWarp`, `malloc`, `realloc`, `strcpy`, `fclose`.
- **Notes:** Line-by-line parsing; skips comment lines (;). Recognizes VERSION, BUTTON_* tokens, and user-defined warp range names.

### SbConfigGetButton
- **Signature:** `char *SbConfigGetButton(char *btnName)`
- **Purpose:** Bidirectional button name lookup: return config name given button name, or button name given config name.
- **Inputs:** Button/config name.
- **Outputs/Return:** Mapped name string, or NULL if not found or empty.
- **Side effects:** None.
- **Calls:** `stricmp`.
- **Notes:** Case-insensitive. Works both directions (e.g., "BUTTON_A" ↔ "MY_BUTTON").

### SbConfigGetButtonNumber
- **Signature:** `int SbConfigGetButtonNumber(char *btnName)`
- **Purpose:** Get zero-based button array index from config name.
- **Inputs:** Button config name.
- **Outputs/Return:** Button index (0–5), or -1 if not found.
- **Side effects:** None.
- **Calls:** `stricmp`.
- **Notes:** Linear search through cfgButtons array.

### SbConfigGetWarpRange
- **Signature:** `WarpRecord *SbConfigGetWarpRange(char *rngName)`
- **Purpose:** Look up named warp range configuration.
- **Inputs:** Warp range name.
- **Outputs/Return:** Pointer to WarpRecord, or NULL if not found.
- **Side effects:** None.
- **Calls:** `stricmp`.
- **Notes:** Linear search through pCfgWarps array.

### SbFxConfigWarp
- **Signature:** `fixed SbFxConfigWarp(WarpRecord *warp, short value)`
- **Purpose:** Apply piecewise-linear warp transformation to input value; return fixed-point result.
- **Inputs:** WarpRecord (or NULL for identity), short input value.
- **Outputs/Return:** Fixed-point output (scaled).
- **Side effects:** None.
- **Calls:** `FIXED_MUL`, `INT_TO_FIXED`, `FIXED_ADD`.
- **Notes:** Iterates through WarpRange array. For each range: if value in [low, high], accumulates (value − low) × multiplier; if value > high, accumulates (high − low) × multiplier. Preserves sign. Returns 0 if warp is NULL (identity). Contains commented-out legacy shift-based approach.

### SbConfigWarp
- **Signature:** `long SbConfigWarp(WarpRecord *warp, short value)`
- **Purpose:** Wrapper around SbFxConfigWarp that converts fixed-point result to integer.
- **Inputs:** WarpRecord, short input value.
- **Outputs/Return:** Integer result (right-shifted by 16 bits).
- **Side effects:** Uses static local variable `r` to work around MSC7.0 compiler bug (stack corruption).
- **Calls:** `SbFxConfigWarp`.
- **Notes:** Static `r` variable is a workaround; result written to static, then extracted after return.

## Control Flow Notes
**Initialization:** `SbConfigParse` is called once to load config file into static state.  
**Query phase:** Runtime code calls `SbConfigGetButton`, `SbConfigGetWarpRange`, etc. to retrieve configuration.  
**Transformation phase:** Input values are warped via `SbFxConfigWarp` or `SbConfigWarp` based on retrieved WarpRecord.  
**No explicit shutdown:** pCfgWarps memory is not freed (leak on program exit, acceptable for DOS-era code).

## External Dependencies
- **Standard C:** `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<ctype.h>`, `<dos.h>`
- **Game/project headers:** `develop.h` (compiler defines), `sbconfig.h` (type definitions), `memcheck.h` (memory debugging)
- **Defined elsewhere:** `strtol`, `stricmp` (non-standard, from RTL), `malloc`, `realloc`, `free`, `fopen`, `fgets`, `fclose`, `strtok`, `strncpy`, `strcpy`, `isspace`
- **Macros from sbconfig.h:** `INT_TO_FIXED`, `FIXED_ADD`, `FIXED_SUB`, `MAX_STRING_LENGTH`
