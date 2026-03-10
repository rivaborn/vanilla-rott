# audiolib/public/pm/source/usrhooks.c

## File Purpose
Provides a modular memory management abstraction layer that wraps standard malloc/free operations. Designed as a "hook" module for library-level memory allocation that calling programs can modify to intercept or customize memory operations.

## Core Responsibilities
- Allocate memory with error checking and return standardized status codes
- Deallocate memory with null-pointer validation
- Provide abstraction point for custom memory management strategies
- Ensure dword-aligned pointer returns (per documentation)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- **Signature:** `int USRHOOKS_GetMem(void **ptr, unsigned long size)`
- **Purpose:** Allocate a block of memory and return its address via output parameter.
- **Inputs:** `size` (number of bytes to allocate)
- **Outputs/Return:** `ptr` (output parameter receiving allocated address), returns `USRHOOKS_Ok` on success or `USRHOOKS_Error` on failure
- **Side effects:** Dynamic memory allocation via `malloc()`; modifies callee-provided pointer
- **Calls:** `malloc(size)`
- **Notes:** Assumes pointer is dword-aligned; returns error code rather than null for failure (caller must check return value)

### USRHOOKS_FreeMem
- **Signature:** `int USRHOOKS_FreeMem(void *ptr)`
- **Purpose:** Deallocate a previously allocated memory block.
- **Inputs:** `ptr` (address to free)
- **Outputs/Return:** `USRHOOKS_Ok` on success, `USRHOOKS_Error` if `ptr` is NULL
- **Side effects:** Dynamic memory deallocation via `free()`
- **Calls:** `free(ptr)`
- **Notes:** Rejects NULL pointers; returns status code rather than void

## Control Flow Notes
Not inferable from this file. This is a utility module providing memory allocation services to the audio library, likely invoked during initialization or on-demand by the audio system.

## External Dependencies
- `stdlib.h` — provides `malloc()`, `free()`
- `usrhooks.h` — defines error enum (`USRHOOKS_Ok`, `USRHOOKS_Error`) and function prototypes
