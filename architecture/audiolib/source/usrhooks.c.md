# audiolib/source/usrhooks.c

## File Purpose
Provides wrapper functions for memory allocation and deallocation, allowing the calling program to customize or intercept dynamic memory operations in the audio library. The module returns standardized error codes rather than raw malloc/free semantics.

## Core Responsibilities
- Wrap malloc/free operations with consistent error reporting
- Abstract memory management behind a defined interface
- Allow callers to override or monitor memory allocation behavior
- Validate allocation requests and return status codes

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- **Signature:** `int USRHOOKS_GetMem(void **ptr, unsigned long size)`
- **Purpose:** Allocate memory and return a pointer to the caller via an output parameter.
- **Inputs:** 
  - `ptr`: Pointer to void pointer (output parameter)
  - `size`: Number of bytes to allocate
- **Outputs/Return:** `USRHOOKS_Ok` (0) on success; `USRHOOKS_Error` (–1) if malloc fails
- **Side effects:** Calls malloc; modifies the memory location referenced by `ptr`
- **Calls:** `malloc()`
- **Notes:** The allocated memory is assumed to be dword-aligned per the inline comment. Uses double-indirection pattern for output (`void **ptr`).

### USRHOOKS_FreeMem
- **Signature:** `int USRHOOKS_FreeMem(void *ptr)`
- **Purpose:** Deallocate memory associated with a pointer.
- **Inputs:** `ptr`: Pointer to memory to free
- **Outputs/Return:** `USRHOOKS_Ok` (0) on success; `USRHOOKS_Error` (–1) if `ptr` is NULL
- **Side effects:** Calls `free()`, deallocating the memory block
- **Calls:** `free()`
- **Notes:** Validates that the pointer is non-NULL before freeing; treats NULL as an error condition.

## Control Flow Notes
Not inferable from this file. These are utility functions called opportunistically by the audio library throughout its lifetime whenever dynamic memory is needed. They are not part of a main init/frame/render cycle.

## External Dependencies
- `stdlib.h` — `malloc()`, `free()`
- `usrhooks.h` — `USRHOOKS_Errors` enum (defines return codes: `USRHOOKS_Ok`, `USRHOOKS_Error`, `USRHOOKS_Warning`)
