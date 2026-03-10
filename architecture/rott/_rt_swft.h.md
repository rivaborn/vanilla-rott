# rott/_rt_swft.h

## File Purpose
Private header for RT_SWIFT.C providing declarations, defines, and structures for the SWIFT input device driver. Manages DOS memory allocation, DPMI real-mode interrupt handling, and device state (Cyberman joystick support).

## Core Responsibilities
- Define interrupt codes (DPMI `0x31`, mouse `0x33`) and device type constants
- Provide x86 register access macros (`AX`, `BX`, `CX`, etc.) for register operations
- Declare global device state (active flag, attached device type)
- Define DPMI real-mode interrupt structure for interrupt context
- Declare DOS memory management functions (allocate/deallocate)
- Declare mouse interrupt handler entry point

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rminfo` | struct | DPMI real-mode interrupt context; holds CPU registers (ax–sp/ss) and flags for interrupt calls |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `fActive` | int | static | Initialization state flag; TRUE after successful init and before termination |
| `nAttached` | int | static | Device type code (SWIFT_DEV_NONE or SWIFT_DEV_CYBERMAN) |
| `regs` | union REGS | global | x86 general-purpose registers (eax, ebx, ecx, edx, esi, edi) |
| `sregs` | struct SREGS | global | x86 segment registers |
| `selector` | short | global | DOS memory block selector (for protected mode) |
| `segment` | short | global | DOS memory block segment (for real mode) |
| `pdosmem` | void far * | global | Far pointer to allocated DOS memory block |
| `RMI` | struct rminfo | static | DPMI real-mode interrupt context structure |

## Key Functions / Methods

### MouseInt
- Signature: `void MouseInt(struct rminfo *prmi)`
- Purpose: Handle mouse interrupt; invoked via DPMI to process mouse input in real mode
- Inputs: Pointer to DPMI real-mode interrupt structure with register context
- Outputs/Return: None
- Side effects: Modifies interrupt context structure; may update device state
- Notes: DPMI entry point called from protected mode to invoke real-mode interrupt handler

### allocDOS
- Signature: `static void far *allocDOS(unsigned nbytes, short *pseg, short *psel)`
- Purpose: Allocate DOS-accessible memory block (low-memory, real-mode accessible)
- Inputs: `nbytes` – size to allocate; `pseg`, `psel` – output pointers for segment/selector
- Outputs/Return: Far pointer to allocated block; writes segment and selector to outputs
- Side effects: Allocates real-mode accessible memory via DPMI
- Notes: Static; caller must store returned pointer and segment/selector for deallocation

### freeDOS
- Signature: `static void freeDOS(short sel)`
- Purpose: Deallocate DOS memory block by selector
- Inputs: `sel` – selector returned from allocDOS
- Outputs/Return: None
- Side effects: Frees allocated DOS memory
- Notes: Static; paired with allocDOS

## Control Flow Notes
Initialization phase driver code. Called during game startup to detect and initialize SWIFT input devices (primarily Cyberman joystick on DOS). Uses DPMI to switch between protected and real modes for device communication via hardware interrupts.

## External Dependencies
- DOS/DPMI-specific: interrupt codes, register structures, real-mode interrupt handling
- x86 inline register macros for CPU register manipulation
- No external symbol dependencies defined in this file
