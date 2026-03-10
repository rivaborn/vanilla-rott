# rott/keyb.h

## File Purpose
Defines a comprehensive mapping of keyboard scan codes to symbolic constant names. Provides a hardware abstraction layer for keyboard input by mapping PS2/DOS era scan codes (hex values) to descriptive identifiers (e.g., `sc_A`, `sc_Return`) used throughout the engine.

## Core Responsibilities
- Define symbolic constants for special keys (modifiers, arrows, function keys)
- Define symbolic constants for alphanumeric keys (letters 0-9)
- Map legacy PC keyboard scan codes to readable names
- Provide compile-time constants for keyboard input handling modules

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This is a pure header file containing compile-time preprocessor constants. It has no control flow. It will be included by keyboard input handling modules (likely in `rott/kb*.c` or similar) that use these constants to map hardware scan codes to game actions.

## External Dependencies
- Standard C preprocessor directives only (`#ifndef`, `#define`, `#endif`)
- No external symbols; purely self-contained definitions

---

### Notes
- Uses legacy PC keyboard scan codes (IBM PS/2 style), typical for DOS-era games
- Maps both printable characters (letters, numbers, punctuation) and special keys (arrows, function keys, modifiers)
- Some redundancies: `sc_Enter` aliases `sc_Return` (0x1c), and `sc_Plus` aliases `sc_Equals` (0x0d)
- Guard `_keyb` prevents multiple inclusion
- Likely used by input capture and remapping systems; scan codes would be translated to game actions elsewhere
