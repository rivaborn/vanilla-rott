# rott/launch.h

## File Purpose
Header defining ProAudio Spectrum (PAS) audio driver interface for legacy DOS systems. Provides BIOS interrupt codes, mixer routing constants, and function declarations for driver detection and invocation using DOS far pointers and x86 register calling conventions.

## Core Responsibilities
- Define PAS driver communication codes (signature, interrupts, command opcodes)
- Define audio mixer input/output channel routing constants
- Declare function pointer table structure for driver audio operations
- Provide driver detection and low-level invocation functions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| MVFunc | struct | Function pointer table for PAS driver operations (SetMixer, SetVolume, SetFilter, GetMixer, GetVolume, GetFilter, GetCrossChannel, ReadSound, FMSplit) |

## Global / File-Static State
None.

## Key Functions / Methods

### PAS_CheckForDriver
- Signature: `int PAS_CheckForDriver( void )`
- Purpose: Detect presence of ProAudio Spectrum audio hardware
- Inputs: None
- Outputs/Return: int (success/failure code)
- Side effects: None inferable
- Calls: Not defined in this file
- Notes: Called during audio subsystem initialization

### PAS_GetFunctionTable
- Signature: `MVFunc far *PAS_GetFunctionTable( void )`
- Purpose: Retrieve pointer to driver operation table
- Inputs: None
- Outputs/Return: Far pointer to MVFunc structure (DOS 16-bit real mode)
- Side effects: None inferable
- Calls: Not defined in this file
- Notes: Far pointer indicates legacy x86 real-mode addressing

### PAS_CallMVFunction
- Signature: `int PAS_CallMVFunction( void ( far *function )( void ), int bx, int cx, int dx )`
- Purpose: Generic low-level driver function invoker using x86 register parameters
- Inputs: function pointer, register values (bx, cx, dx)
- Outputs/Return: int (driver operation result code)
- Side effects: Calls driver function with specified register state
- Calls: Not defined in this file
- Notes: Direct x86 register passing; DOS-era calling convention

## Control Flow Notes
Driver initialization phase: PAS_CheckForDriver() → PAS_GetFunctionTable() → PAS_CallMVFunction() to configure audio hardware. Occurs during engine startup.

## External Dependencies
- Far pointer syntax and x86 register naming indicate 16-bit DOS/legacy code
- MV_* constants are ProAudio Spectrum BIOS interrupt codes (0xbc** range)
- No standard library includes
