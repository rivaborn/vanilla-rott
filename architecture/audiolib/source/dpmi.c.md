# audiolib/source/dpmi.c

## File Purpose

Implements DOS Protected Mode Interface (DPMI) functionality for managing real-mode interrupts and memory locking in a DOS/4GW protected-mode environment. Provides abstraction over DPMI 0x31 interrupt calls to get/set real-mode interrupt vectors, invoke real-mode functions, and lock/unlock memory regions for DMA-safe operations.

## Core Responsibilities

- Get and set real-mode interrupt vectors (DPMI functions 0x0200/0x0201)
- Execute real-mode procedures with register state preservation (DPMI function 0x0301)
- Lock memory regions to prevent paging (DPMI function 0x0600)
- Unlock previously-locked memory regions (DPMI function 0x0601)
- Manage CPU and segment register state for DPMI calls
- Convert pointer addresses to linear addresses for memory locking operations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `dpmi_regs` | struct | Full CPU register state (EAX–SS, flags) for real-mode calls |
| `DPMI_Errors` enum | enum | Error codes (DPMI_Ok, DPMI_Error, DPMI_Warning) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Regs` | `union REGS` | static | CPU register state for DPMI interrupt calls |
| `SegRegs` | `struct SREGS` | static | Segment register state for extended DPMI calls |

## Key Functions / Methods

### DPMI_GetRealModeVector
- **Signature:** `unsigned long DPMI_GetRealModeVector(int num)`
- **Purpose:** Retrieve the current real-mode interrupt vector for a given interrupt number
- **Inputs:** `num` — interrupt number (0–255)
- **Outputs/Return:** 32-bit address (segment:offset packed as `(seg << 16) | offset`)
- **Side effects:** Modifies static `Regs` structure; calls `int386(0x31, ...)`
- **Calls:** `int386()`
- **Notes:** Uses DPMI function 0x0200; extracts CX (segment) and DX (offset) from result

### DPMI_SetRealModeVector
- **Signature:** `void DPMI_SetRealModeVector(int num, unsigned long vector)`
- **Purpose:** Set or replace the real-mode interrupt vector for a given interrupt number
- **Inputs:** `num` — interrupt number; `vector` — 32-bit address (segment:offset packed)
- **Outputs/Return:** void
- **Side effects:** Modifies static `Regs` structure; calls `int386(0x31, ...)` to change interrupt vector in real-mode IVT
- **Calls:** `int386()`
- **Notes:** DPMI function 0x0201; unpacks vector into CX and DX registers

### DPMI_CallRealModeFunction
- **Signature:** `int DPMI_CallRealModeFunction(dpmi_regs *callregs)`
- **Purpose:** Execute a real-mode function/procedure from protected mode, preserving CPU state
- **Inputs:** `callregs` — pointer to `dpmi_regs` structure containing input register values and return frame
- **Outputs/Return:** `DPMI_Ok` (0) on success; `DPMI_Error` (−1) if carry flag set
- **Side effects:** Executes real-mode code; modifies static `Regs` and `SegRegs`; updates `*callregs` with output register state
- **Calls:** `int386x()`, `FP_SEG()`, `FP_OFF()`
- **Notes:** DPMI function 0x0301; uses far pointer macros to pass register block address; check carry flag for errors

### DPMI_LockMemory
- **Signature:** `int DPMI_LockMemory(void *address, unsigned length)`
- **Purpose:** Lock a memory region to prevent virtual memory manager from paging it (required for DMA buffers)
- **Inputs:** `address` — pointer to memory region; `length` — size in bytes
- **Outputs/Return:** `DPMI_Ok` (0) on success; `DPMI_Error` (−1) if carry flag set
- **Side effects:** Modifies static `Regs` structure; calls `int386(0x31, ...)` to lock region in DPMI
- **Calls:** `int386()`
- **Notes:** DPMI function 0x0600; converts pointer to linear address; passes address in BX:CX and length in SI:DI (16-bit pairs)

### DPMI_LockMemoryRegion
- **Signature:** `int DPMI_LockMemoryRegion(void *start, void *end)`
- **Purpose:** Lock memory region specified by start and end pointers (convenience wrapper)
- **Inputs:** `start`, `end` — pointers marking region boundaries
- **Outputs/Return:** `DPMI_Ok` or `DPMI_Error`
- **Side effects:** Calls `DPMI_LockMemory()` with computed length
- **Calls:** `DPMI_LockMemory()`
- **Notes:** Computes length as pointer difference; assumes `end >= start`

### DPMI_UnlockMemory
- **Signature:** `int DPMI_UnlockMemory(void *address, unsigned length)`
- **Purpose:** Unlock a previously-locked memory region, allowing normal paging
- **Inputs:** `address`, `length` — memory region parameters
- **Outputs/Return:** `DPMI_Ok` (0) on success; `DPMI_Error` (−1) if carry flag set
- **Side effects:** Modifies static `Regs`; calls `int386(0x31, ...)` to unlock in DPMI
- **Calls:** `int386()`
- **Notes:** DPMI function 0x0601; mirrors `DPMI_LockMemory()` structure; addresses linear and address/length in register pairs

### DPMI_UnlockMemoryRegion
- **Signature:** `int DPMI_UnlockMemoryRegion(void *start, void *end)`
- **Purpose:** Unlock memory region specified by start and end pointers (convenience wrapper)
- **Inputs:** `start`, `end` — pointers marking region boundaries
- **Outputs/Return:** `DPMI_Ok` or `DPMI_Error`
- **Side effects:** Calls `DPMI_UnlockMemory()` with computed length
- **Calls:** `DPMI_UnlockMemory()`
- **Notes:** Mirrors `DPMI_LockMemoryRegion()` pattern

## Control Flow Notes

This file provides low-level DPMI support for audio initialization and runtime management. Not part of the main frame/update loop. Called during:
- **Init**: `DPMI_LockMemory()` called to lock audio buffers for DMA; real-mode interrupt vectors set via `DPMI_SetRealModeVector()`
- **Shutdown**: Audio memory unlocked via `DPMI_UnlockMemory()`
- **Runtime**: Real-mode function calls invoked via `DPMI_CallRealModeFunction()` (if audio driver uses real-mode callbacks)

## External Dependencies

- **`<dos.h>`** – `union REGS`, `struct SREGS`, `int386()`, `int386x()`, `FP_SEG()`, `FP_OFF()` macros for register and interrupt manipulation
- **`<string.h>`** – included but not used in this file
- **`dpmi.h`** – local header defining `dpmi_regs` struct, `DPMI_Errors` enum, and function prototypes
- **DPMI BIOS interrupt 0x31** – invoked via `int386()` and `int386x()` for DPMI services
