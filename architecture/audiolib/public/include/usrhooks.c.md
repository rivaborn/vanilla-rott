# audiolib/public/include/usrhooks.c

## File Purpose
Provides wrapper functions for memory allocation and deallocation operations that the audio library requires. These functions are intentionally left public and modifiable so the calling program can customize or restrict memory operations as needed.

## Core Responsibilities
- Wrap standard memory allocation (malloc) with error handling and return code semantics
- Wrap standard memory deallocation (free) with validation and return code semantics
- Provide a customization point for the library caller to override memory management behavior
- Maintain interface compatibility through function prototypes in the header

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- **Signature:** `int USRHOOKS_GetMem(void **ptr, unsigned long size)`
- **Purpose:** Allocate memory and return a pointer via output parameter, with error reporting.
- **Inputs:** `size` — number of bytes to allocate
- **Outputs/Return:** `ptr` — output parameter filled with allocated pointer (on success); return code (`USRHOOKS_Ok` or `USRHOOKS_Error`)
- **Side effects (global state, I/O, alloc):** Calls `malloc()` to allocate heap memory
- **Calls (direct calls visible in this file):** `malloc()`
- **Notes:** Returns error code if malloc fails. Assumes returned pointer is dword-aligned (comment in code). Uses output parameter pattern instead of direct return to avoid pointer-sized return values.

### USRHOOKS_FreeMem
- **Signature:** `int USRHOOKS_FreeMem(void *ptr)`
- **Purpose:** Deallocate memory with null-pointer validation.
- **Inputs:** `ptr` — pointer to memory to free
- **Outputs/Return:** Return code (`USRHOOKS_Ok` or `USRHOOKS_Error`)
- **Side effects (global state, I/O, alloc):** Calls `free()` to deallocate heap memory
- **Calls (direct calls visible in this file):** `free()`
- **Notes:** Rejects NULL pointers with error code rather than passing them to free. Returns error rather than void to enable caller error handling.

## Control Flow Notes
These are hook/wrapper functions called by the audio library whenever it needs to allocate or free memory. They are not part of a frame loop; they are utility functions meant to be customizable entry points. The calling program may override these implementations to add tracking, pooling, or memory restrictions.

## External Dependencies
- **stdlib.h** — standard C library (malloc, free)
- **usrhooks.h** — local header defining error codes and function prototypes
