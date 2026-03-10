# rott/rt_vh_a.h

## File Purpose
Public header declaring interface functions to assembly-language video hardware and input handling code. Declares screen update and joystick input functions with register preservation constraints via compiler pragma.

## Core Responsibilities
- Declare screen update/refresh entry point (`VH_UpdateScreen`)
- Declare joystick input reading entry point (`JoyStick_Vals`)
- Specify register clobbering information for assembly functions via `#pragma aux`

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### VH_UpdateScreen
- Signature: `void VH_UpdateScreen(void)`
- Purpose: Update or refresh the video display/screen buffer
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies video hardware state or framebuffer
- Calls: Implemented in `rt_vh_a.asm` (not visible here)
- Notes: Entry point for display refresh; likely called once per frame

### JoyStick_Vals
- Signature: `void JoyStick_Vals(void)`
- Purpose: Read current joystick input state and populate registers with values
- Inputs: None
- Outputs/Return: Values returned in CPU registers (`eax`, `ebx`, `ecx`, `edx`, `esi`, `edi`)
- Side effects: Reads hardware input; modifies all general-purpose registers
- Calls: Implemented in `rt_vh_a.asm` (not visible here)
- Notes: The `#pragma aux` directive tells the compiler this function modifies all six general-purpose registers; caller must not assume register values are preserved

## Control Flow Notes
Fits into the main frame update loop: `VH_UpdateScreen()` likely called during render phase; `JoyStick_Vals()` called during input poll phase. Both are low-level hardware interface routines.

## External Dependencies
- Implementation in `rt_vh_a.asm` (assembly language module)
- Uses Watcom C pragmas (`#pragma aux`) for register-level calling conventions
