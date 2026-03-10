# audiolib/public/timer/source/usrhooks.c

## File Purpose
Provides wrapper functions for memory allocation and deallocation that serve as customization points for the audio library. The module allows the calling program to intercept or restrict memory operations while maintaining a consistent interface.

## Core Responsibilities
- Allocate dynamic memory with error checking
- Deallocate dynamic memory with validation
- Return standardized error codes (Ok/Error)
- Abstract malloc/free behind a library-controlled interface

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- **Signature:** `int USRHOOKS_GetMem(void **ptr, unsigned long size)`
- **Purpose:** Allocate memory of specified size and return pointer via output parameter.
- **Inputs:** `size` (bytes to allocate), `ptr` (output parameter for allocated pointer)
- **Outputs/Return:** Status code: `USRHOOKS_Ok` on success, `USRHOOKS_Error` on failure; allocated pointer written to `*ptr`
- **Side effects:** Calls `malloc()` to allocate heap memory; may set `*ptr` to NULL on allocation failure
- **Calls:** `malloc()` (C standard library)
- **Notes:** Returns error code if `malloc()` returns NULL; pointer is assumed to be dword-aligned (per header comment)

### USRHOOKS_FreeMem
- **Signature:** `int USRHOOKS_FreeMem(void *ptr)`
- **Purpose:** Deallocate previously allocated memory.
- **Inputs:** `ptr` (pointer to free)
- **Outputs/Return:** Status code: `USRHOOKS_Ok` on success, `USRHOOKS_Error` if ptr is NULL
- **Side effects:** Calls `free()` to deallocate heap memory
- **Calls:** `free()` (C standard library)
- **Notes:** Validates that `ptr` is not NULL before freeing; errors on NULL input (defensive check)

## Control Flow Notes
These functions are hook points for the audio library's memory management. They are called during initialization (allocating audio buffers, data structures) and shutdown (cleanup) phases. The wrapping allows callers to substitute custom allocators or enforce memory constraints.

## External Dependencies
- **Includes:** `<stdlib.h>` (malloc, free), `usrhooks.h` (local error code definitions)
- **Defined elsewhere:** All caller dependencies on `USRHOOKS_GetMem()` and `USRHOOKS_FreeMem()` throughout the audio library
