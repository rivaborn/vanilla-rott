# audiolib/source/dpmi.h

## File Purpose
Header providing DPMI (DOS Protected Mode Interface) wrappers for low-level x86 DOS operations. Enables protected-mode code to interact with real-mode DOS functionality, including memory management, interrupt vectors, and real-mode function calls. Designed for DOS extender environments (e.g., DOS/4GW).

## Core Responsibilities
- Define DPMI error codes and register state structure
- Declare functions for DOS memory allocation and deallocation
- Provide memory locking/unlocking utilities for DMA-safe regions
- Offer real-mode interrupt vector manipulation
- Enable calling real-mode DOS functions from protected mode
- Implement low-level operations via x86 inline assembly (pragma aux)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| DPMI_Errors | enum | Error return codes: Warning (-2), Error (-1), Ok (0) |
| dpmi_regs | struct | x86 register state (EDI, ESI, EBP, EBX, EDX, ECX, EAX, Flags, segment regs, CS:IP, SS:SP) |

## Global / File-Static State
None.

## Key Functions / Methods

### DPMI_GetRealModeVector
- **Signature:** `unsigned long DPMI_GetRealModeVector(int num)`
- **Purpose:** Retrieve the address of a real-mode interrupt handler.
- **Inputs:** Interrupt vector number.
- **Outputs/Return:** Real-mode address (offset/segment packed in one dword).
- **Side effects:** None.
- **Calls:** None visible; implemented via pragma aux assembly.
- **Notes:** Used to save/hook DOS interrupt handlers during init.

### DPMI_CallRealModeFunction
- **Signature:** `int DPMI_CallRealModeFunction(dpmi_regs *callregs)`
- **Purpose:** Execute a real-mode function with specified register state, switching CPU mode as needed.
- **Inputs:** Pointer to dpmi_regs with EAX, EBX, etc. pre-set.
- **Outputs/Return:** DPMI_Errors code; registers updated in callregs after return.
- **Side effects:** CPU mode switch; modifies all x86 registers; may trigger DMA operations.
- **Calls:** x86 int 31h (DPMI interrupt).
- **Notes:** Used for DOS calls (e.g., allocating conventional memory).

### DPMI_GetDOSMemory
- **Signature:** `int DPMI_GetDOSMemory(void **ptr, int *descriptor, unsigned length)`
- **Purpose:** Allocate a block of conventional (DOS) memory and create a protected-mode selector.
- **Inputs:** ptr (output buffer), descriptor (output handle), length (bytes).
- **Outputs/Return:** DPMI_Errors; ptr and descriptor written on success.
- **Side effects:** Allocates DOS memory; creates LDT descriptor.
- **Calls:** x86 int 31h (DPMI function 0x0100).
- **Notes:** Implemented via pragma aux. Output ptr is aligned to paragraph boundary.

### DPMI_FreeDOSMemory
- **Signature:** `int DPMI_FreeDOSMemory(int descriptor)`
- **Purpose:** Release previously allocated DOS memory block.
- **Inputs:** Descriptor handle from DPMI_GetDOSMemory.
- **Outputs/Return:** DPMI_Errors code.
- **Side effects:** Frees DOS memory; invalidates descriptor.
- **Calls:** x86 int 31h (DPMI function 0x0101).
- **Notes:** Implemented via pragma aux.

### DPMI_LockMemory / DPMI_UnlockMemory
- **Signature:** `int DPMI_LockMemory(void *address, unsigned length)` / `int DPMI_UnlockMemory(...)`
- **Purpose:** Lock/unlock memory region to prevent paging (required for DMA operations).
- **Inputs:** Address and byte length of region.
- **Outputs/Return:** DPMI_Errors code.
- **Side effects:** Pins pages into physical RAM or releases pins.
- **Calls:** Not visible (declaration only).
- **Notes:** Called before DMA-based I/O; companion macros DPMI_Lock(var), DPMI_Unlock(var) provided.

### DPMI_LockMemoryRegion / DPMI_UnlockMemoryRegion
- **Signature:** `int DPMI_LockMemoryRegion(void *start, void *end)` / `int DPMI_UnlockMemoryRegion(...)`
- **Purpose:** Lock/unlock a range of memory between two pointers.
- **Inputs:** start and end pointers (typically stack base/top, heap, etc.).
- **Outputs/Return:** DPMI_Errors code.
- **Side effects:** Pins/unpins memory region.
- **Calls:** Not visible (declaration only).
- **Notes:** Convenience variant of the byte-length versions.

## Control Flow Notes
- **Init/Shutdown context:** Used during audio library initialization to allocate DMA buffers in DOS memory and lock them to prevent paging. Real-mode vector calls may appear during ISR setup.
- **Frame/Update/Render:** Not called during normal frame operation; only during setup/teardown.
- **x86 assembly blocks:** pragma aux directives compile inline x86 code directly into functions (Watcom C syntax). DPMI_GetDOSMemory and DPMI_FreeDOSMemory have complete implementations; others are declarations only.

## External Dependencies
- **x86 CPU:** int 31h (DPMI interrupt).
- **DOS Extender (e.g., DOS/4GW):** Provides DPMI services.
- **Watcom C compiler:** pragma aux syntax for inline assembly.
- No external headers included.
