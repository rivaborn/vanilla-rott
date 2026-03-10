# audiolib/source/irq.c

## File Purpose
Low-level DOS Protected Mode Interface (DPMI) interrupt handler management. Bridges between real mode and protected mode interrupt handlers, allowing safe interrupt vector setup and restoration in a DOS extender environment (1994–1995 Apogee era).

## Core Responsibilities
- Allocate DOS conventional memory for real mode interrupt stubs via DPMI function 0x0100
- Set up DPMI real mode callbacks (function 0x0303) to translate real mode → protected mode calls
- Install protected and real mode interrupt vectors via DPMI functions 0x0205 and 0x0201
- Restore original interrupt handlers on cleanup
- Manage callback registration/deregistration through DPMI functions 0x0304

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `DPMI_REGS` | struct | x86 register state for DPMI calls (32-bit EAX/EBX/ECX/EDX/ESI/EDI/EBP plus flags and segment registers) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rmregs` | `DPMI_REGS` | static | Register state passed to DPMI real mode callback |
| `IRQ_Callback` | function pointer (`__interrupt __far`) | static | User-supplied protected mode interrupt handler |
| `IRQ_RealModeCode` | `char *` | static | Pointer to 6-byte DOS memory stub (allocated via DPMI) containing real mode interrupt code |
| `IRQ_CallBackSegment`, `IRQ_CallBackOffset` | `unsigned short` | static | DPMI callback address (returned by function 0x0303) |
| `IRQ_RealModeSegment`, `IRQ_RealModeOffset` | `unsigned short` | static | Original real mode handler address (saved for restoration) |
| `IRQ_ProtectedModeSelector`, `IRQ_ProtectedModeOffset` | `unsigned short`, `unsigned long` | static | Original protected mode handler address (saved for restoration) |
| `Regs`, `SegRegs` | `union REGS`, `struct SREGS` | static | Registers for int386/int386x DPMI calls |

## Key Functions / Methods

### D32DosMemAlloc
- **Signature:** `static void *D32DosMemAlloc(unsigned long size)`
- **Purpose:** Allocate contiguous DOS conventional memory via DPMI function 0x0100
- **Inputs:** `size` — number of bytes to allocate
- **Outputs/Return:** Linear address of allocated block; `0` on failure (carry flag set)
- **Side effects:** Modifies `Regs.x.eax`, `Regs.x.ebx`; calls int386 DPMI interrupt 0x31
- **Calls:** `int386`
- **Notes:** Converts size to paragraph units (16 bytes), shifts result left 4 bits to get linear address. Used to allocate 6-byte low-memory stub.

### fixebp
- **Signature:** `void fixebp(void)` (inline assembly, Watcom `#pragma aux`)
- **Purpose:** Fix EBP and ESP for 16-bit stack context when called from DPMI callback
- **Inputs:** None (examines SS and current stack frame size via LAR instruction)
- **Outputs/Return:** None (modifies EBP and ESP)
- **Side effects:** Checks SS descriptor bit 22 (big stack bit); zero-extends SP if 16-bit stack detected
- **Notes:** Ensures 32-bit stack operations work correctly in mixed 16/32-bit transition

### rmcallback
- **Signature:** `void rmcallback(unsigned short _far *stkp)`
- **Purpose:** Intermediary C function; extracts real mode return frame from stack and calls protected mode handler
- **Inputs:** `stkp` — far pointer to real mode stack containing return address
- **Outputs/Return:** None
- **Side effects:** Populates `rmregs.drip`, `rmregs.drcs`, `rmregs.drsp`; calls `IRQ_Callback()`
- **Calls:** `IRQ_Callback()`
- **Notes:** Reconstructs the real mode return frame by popping IP and CS from the incoming stack pointer

### callback_x
- **Signature:** `static void _interrupt _cdecl callback_x(int rgs, int rfs, int res, int rds, int rdi, int rsi, int rbp, int rsp, int rbx, int rdx, int rcx, int rax)`
- **Purpose:** Protected mode interrupt handler entry point called by DPMI callback; bridges to rmcallback
- **Inputs:** Register values pushed by interrupt prologue (registers in symbolic parameter order)
- **Outputs/Return:** None
- **Side effects:** Calls `fixebp()` to adjust stack context, then calls `rmcallback()`
- **Calls:** `fixebp()`, `rmcallback()`
- **Notes:** Parameters are named to match x86 register names (rax, rbx, rcx, etc.) but are actually the values pushed by the interrupt prologue. Uses MK_FP macro to construct far pointer from DS:SI.

### IRQ_SetVector
- **Signature:** `int IRQ_SetVector(int vector, void (__interrupt __far *function)(void))`
- **Purpose:** Install a new interrupt handler and save original handler addresses
- **Inputs:** `vector` — IRQ number (0–15); `function` — protected mode interrupt handler
- **Outputs/Return:** `IRQ_Ok` on success, `IRQ_Error` on failure
- **Side effects:** Allocates DOS memory if not yet done; installs 6-byte real mode stub; modifies interrupt vectors via DPMI; saves original handlers in static state
- **Calls:** `int386`, `int386x`, `D32DosMemAlloc`
- **Notes:** Sequence: (1) Save original PM/RM vectors via DPMI 0x0204/0x0200, (2) Allocate DPMI callback via 0x0303, (3) Allocate low-memory code stub (if first call), (4) Write x86 code (CALL FAR + IRET) into stub, (5) Install PM handler via 0x0205, (6) Install RM handler via 0x0201. The real mode stub performs a far call to the DPMI callback.

### IRQ_RestoreVector
- **Signature:** `int IRQ_RestoreVector(int vector)`
- **Purpose:** Restore original interrupt handlers and free DPMI callback resources
- **Inputs:** `vector` — IRQ number to restore
- **Outputs/Return:** `IRQ_Ok` on success, `IRQ_Error` if DPMI callback free fails
- **Side effects:** Restores real mode vector via DPMI 0x0201, protected mode vector via 0x0205, frees callback via 0x0304
- **Calls:** `int386`, `int386x`
- **Notes:** Uses saved addresses from IRQ_SetVector static state. Does NOT free the allocated low-memory code stub (IRQ_RealModeCode).

## Control Flow Notes
This module is **initialization/shutdown only**—not part of frame/update/render loops. Typical call sequence:
1. **Init:** `IRQ_SetVector(IRQ_NUM, handler_func)` during sound subsystem startup
2. **Runtime:** Interrupt fires → real mode stub → DPMI callback → `callback_x()` → `rmcallback()` → `IRQ_Callback()` (user handler)
3. **Shutdown:** `IRQ_RestoreVector(IRQ_NUM)` to restore original handlers

## External Dependencies
- **Includes:** `<dos.h>` (int386, int386x, REGS, SREGS, FP_SEG, FP_OFF, MK_FP), `<stdlib.h>` (unused), `irq.h` (local header with error enum and function declarations)
- **DPMI services:** Interrupt 0x31 (DPMI) with functions 0x0100 (allocate DOS memory), 0x0200/0x0201 (get/set RM vector), 0x0204/0x0205 (get/set PM vector), 0x0303 (allocate callback), 0x0304 (free callback)
- **Macros from dos.h:** `FP_SEG()`, `FP_OFF()`, `MK_FP()` for far pointer manipulation
