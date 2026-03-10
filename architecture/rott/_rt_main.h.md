# rott/_rt_main.h

## File Purpose
Private header for the main game loop subsystem. Declares core loop functions (`GameLoop`, `PlayLoop`), keyboard polling, screen capture utilities (LBM/PCX writers), and related bitmap format structures for the ROTT engine.

## Core Responsibilities
- Game loop initialization and execution entry points
- Keyboard input polling interface
- Color palette management
- Screen-to-file capture (LBM and PCX formats) with conditional compilation
- Bitmap file format definitions (BMHD/IFF, PCX)
- Quick-load state checking
- Time constants for quit behavior

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `bmhd_t` | struct | IFF BMHD (bitmap header) for Amiga-format image files; stores dimensions, bit depth, compression, transparency |
| `PCX_HEADER` | struct | PCX image file header; encodes dimensions, color palette (16×3), bit depth, line spacing |

## Global / File-Static State
None.

## Key Functions / Methods

### GameLoop
- Signature: `void GameLoop(void)`
- Purpose: Main game loop entry point; orchestrates the overall game state machine
- Inputs: None
- Outputs/Return: None
- Side effects: Drives all game logic, rendering, and input polling; infinite loop until game exit
- Calls: Not visible in this file (defined elsewhere)
- Notes: Core execution loop; likely calls `PlayLoop` internally

### PlayLoop
- Signature: `void PlayLoop(void)`
- Purpose: Secondary/gameplay-specific loop; likely invoked from `GameLoop` during active gameplay
- Inputs: None
- Outputs/Return: None
- Side effects: Executes frame logic during active play
- Calls: Not visible in this file
- Notes: Presumably nested or conditional within `GameLoop`

### PollKeyboard
- Signature: `void PollKeyboard(void)`
- Purpose: Service keyboard input events; update input state for the current frame
- Inputs: None
- Outputs/Return: None
- Side effects: Updates global/engine input state
- Calls: Not visible in this file
- Notes: Likely called once per frame by `GameLoop` or `PlayLoop`

### FixColorMap
- Signature: `void FixColorMap(void)`
- Purpose: Adjust or normalize the active color palette
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies global palette state
- Calls: Not visible in this file
- Notes: Timing/frequency unknown

### CheckForQuickLoad
- Signature: `boolean CheckForQuickLoad(void)`
- Purpose: Poll for a quick-load input/state
- Inputs: None
- Outputs/Return: Boolean (true if quick-load triggered)
- Side effects: May reset game state if quick-load confirmed
- Calls: Not visible in this file
- Notes: Used to restore previously saved game state on demand

### WriteLBMfile
- Signature: `void WriteLBMfile(char *filename, byte *data, int width, int height)`
- Purpose: Write framebuffer/image data to LBM (IFF) format file for screenshots
- Inputs: Filename (string), image data (byte array), width, height in pixels
- Outputs/Return: None (writes file)
- Side effects: Disk I/O; creates or overwrites file
- Calls: Not visible in this file
- Notes: Conditional on `SAVE_SCREEN` macro; uses `bmhd_t` header format

### WritePCX
- Signature: `void WritePCX(char *file, byte *source)`
- Purpose: Write image data to PCX format file
- Inputs: Filename, image data buffer
- Outputs/Return: None (writes file)
- Side effects: Disk I/O; creates or overwrites file
- Calls: `PutBytes` (visible in this file)
- Notes: Conditional on `SAVE_SCREEN`; helper for screenshot capture

### PutBytes
- Signature: `int PutBytes(unsigned char *ptr, unsigned int bytes)`
- Purpose: Write a byte sequence to output (likely PCX file)
- Inputs: Byte pointer, byte count
- Outputs/Return: Count of bytes written (or status code)
- Side effects: Disk I/O
- Calls: Not visible in this file
- Notes: Helper for `WritePCX`; likely manages file buffering or RLE encoding

### GetFileName
- Signature: `void GetFileName(boolean saveLBM)`
- Purpose: Prompt user for/construct filename for screenshot save
- Inputs: Boolean flag (true = LBM format, false = PCX format)
- Outputs/Return: None (sets global filename or similar)
- Side effects: User interaction (menu/dialog); may modify global state
- Calls: Not visible in this file
- Notes: Conditional on `SAVE_SCREEN`

### DrawRottTitle
- Signature: `void DrawRottTitle(void)`
- Purpose: Render the ROTT title screen/splash
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies framebuffer/display
- Calls: Not visible in this file
- Notes: Conditional on `SAVE_SCREEN`; likely called during startup or main menu

## Control Flow Notes
This header exposes the core frame-loop interface. `GameLoop()` is the top-level entry point; it likely calls `PlayLoop()` for active gameplay and `PollKeyboard()` each frame. Keyboard, input, and palette management are integrated into the loop. Screen capture (LBM/PCX) is a conditional feature (gated by `SAVE_SCREEN` macro) for debug/demo purposes.

## External Dependencies
- **Includes**: `develop.h` (feature/debug flags)
- **Types used**: `void`, `boolean`, `byte`, `char`, `int`, `unsigned char`, `unsigned short`, `short` (all C primitives; definitions elsewhere)
- **Macros**: `QUITTIMEINTERVAL`, `SAVE_SCREEN` (conditional compilation flag)
- **Symbols defined elsewhere**: `develop.h` defines all feature flags (`SHAREWARE`, `SUPERROTT`, etc.) and debug modes
