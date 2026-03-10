# rott/sbconfig.h

## File Purpose
Configuration header for Sound Blaster button mappings and warp range (input value scaling) settings. Defines the data structures and public API for parsing `.cfg` files and retrieving button/warp configurations at runtime.

## Core Responsibilities
- Define `WarpRange` and `WarpRecord` structures for value mapping/scaling
- Parse configuration files with VERSION, BUTTON, and RANGE entries
- Provide lookup functions for button name mappings (bidirectional)
- Provide lookup functions for named warp range configurations
- Support value warping (scaling) using fixed-point arithmetic
- Define lexical/syntax rules for configuration file format

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `WarpRange` | struct | Maps an input range `[low, high]` to a fixed-point multiplier for scaling values |
| `WarpRecord` | struct | Associates a named warp configuration with an array of `WarpRange` entries |

## Global / File-Static State
None.

## Key Functions / Methods

### SbConfigParse
- Signature: `int SbConfigParse(char *filename)`
- Purpose: Parse the configuration file and load button/warp settings into internal state
- Inputs: `filename` – path to configuration file
- Outputs/Return: Integer status code (likely 0 for success)
- Side effects: Loads and caches configuration in global/static state
- Calls: (implementation not visible)
- Notes: Called once at initialization; subsequent queries use cached data

### SbConfigGetButton
- Signature: `char *SbConfigGetButton(char *btnName)`
- Purpose: Bidirectional button name lookup (e.g., "BUTTON_A" ↔ "MY_BUTTON")
- Inputs: `btnName` – button name (either physical or mapped name)
- Outputs/Return: Mapped button name or NULL if not found
- Side effects: None
- Calls: (implementation not visible)
- Notes: Case-insensitive; makes physical button names (BUTTON_A, etc.) reserved

### SbConfigGetButtonNumber
- Signature: `int SbConfigGetButtonNumber(char *btnName)`
- Purpose: Map button name to numeric identifier
- Inputs: `btnName` – button name
- Outputs/Return: Button number or error code
- Side effects: None
- Calls: (implementation not visible)

### SbConfigGetWarpRange
- Signature: `WarpRecord *SbConfigGetWarpRange(char *rngName)`
- Purpose: Retrieve warp configuration by name
- Inputs: `rngName` – warp range name
- Outputs/Return: Pointer to `WarpRecord` or NULL if not found
- Side effects: None
- Calls: (implementation not visible)

### SbFxConfigWarp / SbConfigWarp
- Signature: `fixed SbFxConfigWarp(WarpRecord *warp, short value)` / `long SbConfigWarp(WarpRecord *warp, short value)`
- Purpose: Apply warp (scale) transformation to an input value based on range membership
- Inputs: `warp` – warp configuration, `value` – input value to warp
- Outputs/Return: Scaled value (fixed-point or integer)
- Side effects: None
- Calls: (implementation not visible)
- Notes: Selects appropriate multiplier from `WarpRange` based on which range `value` falls into

## Macro Utilities
- `INT_TO_FIXED`, `FIXED_TO_INT`, `FLOAT_TO_FIXED`: Convert between fixed-point and integer/float
- `FIXED_ADD`, `FIXED_SUB`: Fixed-point arithmetic helpers

## Control Flow Notes
This is a **configuration module** invoked at startup:
1. Game/engine calls `SbConfigParse(filename)` once to load settings
2. Throughout runtime, code queries button and warp settings via the lookup functions
3. Warp functions are called to apply input scaling (e.g., joystick sensitivity adjustment)

## External Dependencies
- `fixed` typedef (defined elsewhere, likely a fixed-point type)
- Configuration file format is custom (see syntax diagram at bottom of file)
