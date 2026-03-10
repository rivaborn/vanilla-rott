# rottcom/rottser/keyb.h

## File Purpose
Header file defining keyboard scan code constants for the Rise of the Triad engine. Maps physical keyboard keys (at AT/PS2 hardware level) to named symbolic constants used throughout input event handling.

## Core Responsibilities
- Define symbolic constants for all supported keyboard scan codes
- Provide named identifiers for input event processing (e.g., `sc_Return`, `sc_Escape`)
- Map special keys (function keys, arrows, modifiers, navigation)
- Map alphanumeric keys (A–Z, 0–9)
- Supply fallback/sentinel values (`sc_None`, `sc_Bad`)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This is a utility header included by input/keyboard modules. During keyboard event processing, hardware scan codes received from the OS/driver are compared against these constants to identify which logical key was pressed. The constants act as an abstraction layer between raw hardware codes and engine gameplay logic.

## External Dependencies
- None (self-contained constants file)

---

**Notes:**
- All macros use the `sc_` prefix (likely "scan code")
- Scan codes match standard IBM AT/PS2 keyboard protocol
- `sc_Enter` is aliased to `sc_Return` (0x1c)
- `sc_Bad` (0xff) and `sc_None` (0x00) serve as error/no-key sentinel values
- No conditional compilation or platform-specific branching visible
