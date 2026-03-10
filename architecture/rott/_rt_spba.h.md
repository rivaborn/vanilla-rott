# rott/_rt_spba.h

## File Purpose
Private header for spaceball input device support in ROTT. Defines the sign-extraction macro and the count of spaceball buttons (6 buttons on the hardware device).

## Core Responsibilities
- Define the `SGN()` macro for extracting the sign of a numeric value (1 if positive, -1 if negative)
- Declare the spaceball button count as a compile-time constant
- Serve as a private include guard to prevent redefinition of spaceball-related constants

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `NUMSPACEBALLBUTTONS` | Preprocessor constant (int literal) | File-static | Specifies the number of buttons on a spaceball input device (legacy 3D mouse) |

## Key Functions / Methods
None. This file contains only preprocessor macros and constants.

**Notes:**
- The `SGN(x)` macro returns 1 for positive values and -1 for non-positive values (note: returns -1 for both zero and negative, not a standard three-way sign function).

## Control Flow Notes
Not inferable. This is a configuration/definition header likely included by spaceball input handling code during initialization or input polling routines.

## External Dependencies
- No external includes or symbols; this is self-contained.
- The `_rt_spba_private` guard suggests it is private to the spaceball subsystem and should not be included by external code.
