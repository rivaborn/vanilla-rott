# rott/usrhooks.c

## File Purpose
Provides a thin wrapper layer for memory allocation and deallocation that the game engine library requires. Allows customization of how the engine obtains and releases memory by delegating to the Z_Zone memory manager. Explicitly designed as a public hook point for modification.

## Core Responsibilities
- Allocate memory on behalf of library code and return success/error status
- Deallocate memory safely with NULL-pointer validation
- Serve as abstraction layer between library requests and the Z_Zone manager
- Enable user customization of memory handling behavior

## Key Types / Data Structures
None defined in this file (enum USRHOOKS_Errors defined in usrhooks.h).

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- **Signature:** `int USRHOOKS_GetMem(void **ptr, unsigned long size)`
- **Purpose:** Allocate memory and return status to caller via output parameter.
- **Inputs:** `ptr` (output pointer to store allocated memory), `size` (bytes to allocate)
- **Outputs/Return:** `int` — `USRHOOKS_Ok` (0) on success, `USRHOOKS_Error` (−1) if allocation fails
- **Side effects:** Calls Z_Malloc with PU_STATIC tag (memory persists entire execution); writes allocated pointer to `*ptr`
- **Calls:** `Z_Malloc`
- **Notes:** Assumes Z_Malloc returns dword-aligned pointers; NULL result indicates failure

### USRHOOKS_FreeMem
- **Signature:** `int USRHOOKS_FreeMem(void *ptr)`
- **Purpose:** Deallocate memory associated with a pointer.
- **Inputs:** `ptr` (pointer to deallocate)
- **Outputs/Return:** `int` — `USRHOOKS_Ok` (0) on success, `USRHOOKS_Error` (−1) if `ptr == NULL`
- **Side effects:** Calls Z_Free to return memory to heap; validates pointer is non-NULL first
- **Calls:** `Z_Free`
- **Notes:** Returns error code if NULL pointer passed; otherwise always succeeds (Z_Free does not report failure)

## Control Flow Notes
These are support functions called during engine initialization and runtime whenever a subsystem needs to allocate memory. Not part of the frame loop. The use of PU_STATIC tag ensures allocated memory survives until shutdown.

## External Dependencies
- **z_zone.h** — Z_Malloc, Z_Free function declarations; PU_STATIC memory tag constant
- **memcheck.h** — Memory debugging instrumentation (compiles out if NOMEMCHECK defined)
