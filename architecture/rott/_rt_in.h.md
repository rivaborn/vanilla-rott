# rott/_rt_in.h

## File Purpose
Private header for RT_IN.C input handling module. Defines low-level input device constants, ISR numbers, and an inline x86 assembly function for BIOS mouse interrupt control in a DOS/legacy environment.

## Core Responsibilities
- Define ISR and BIOS interrupt numbers for keyboard, mouse, and joystick input
- Specify hardware control constants for mouse subcommands and joystick scaling
- Declare and implement low-level mouse control via inline x86 assembly

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### Mouse
- **Signature:** `void Mouse (int x);`
- **Purpose:** Invoke BIOS mouse interrupt (INT 33h) with a command/parameter code.
- **Inputs:** `x` (int) – command code, loaded into EAX register before interrupt.
- **Outputs/Return:** void
- **Side effects:** Executes x86 BIOS interrupt; modifies EAX register; may alter mouse state or retrieve mouse status depending on command.
- **Calls:** INT 33h (BIOS mouse interrupt)
- **Notes:** Implemented via `#pragma aux` inline assembly (Watcom C syntax). Assumes real-mode DOS or compatible BIOS environment. The specific command in `x` determines behavior (reset, query buttons, etc.).

## Control Flow Notes
This header is likely used during engine initialization to set up input handlers and during the main game loop for polling mouse state. It represents the lowest abstraction layer before higher-level input polling/event functions in RT_IN.C.

## External Dependencies
- No local includes
- External: x86 BIOS interrupts (INT 9 for keyboard, INT 0x33 for mouse)
- Assumes: Watcom C compiler, real-mode x86 architecture, DOS environment
