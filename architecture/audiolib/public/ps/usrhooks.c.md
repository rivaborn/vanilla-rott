# audiolib/public/ps/usrhooks.c

## File Purpose
Provides memory allocation and deallocation wrapper functions for the audio library. This module allows the calling program to intercept or customize memory operations required by the audio subsystem. The functions are intentionally left public for user modification.

## Core Responsibilities
- Wrap dynamic memory allocation via `malloc`
- Wrap dynamic memory deallocation via `free`
- Return standard error codes (`USRHOOKS_Ok`, `USRHOOKS_Error`) to indicate success/failure
- Ensure allocated memory satisfies dword-alignment requirement
- Validate pointers before deallocation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `USRHOOKS_Errors` | enum | Status codes for allocation/deallocation results |

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- **Signature:** `int USRHOOKS_GetMem(void **ptr, unsigned long size)`
- **Purpose:** Allocate a block of dynamic memory of the requested size.
- **Inputs:** `ptr` (output pointer to store allocated address), `size` (bytes to allocate)
- **Outputs/Return:** Returns `USRHOOKS_Ok` on success or `USRHOOKS_Error` on failure; sets `*ptr` to allocated address (or leaves it unchanged on failure).
- **Side effects:** Calls `malloc`, which allocates heap memory.
- **Calls:** `malloc`
- **Notes:** Allocated memory is assumed to be dword-aligned by caller. Returns error if `malloc` returns NULL.

### USRHOOKS_FreeMem
- **Signature:** `int USRHOOKS_FreeMem(void *ptr)`
- **Purpose:** Deallocate a previously allocated memory block.
- **Inputs:** `ptr` (address of block to free)
- **Outputs/Return:** Returns `USRHOOKS_Ok` on success or `USRHOOKS_Error` if pointer is NULL.
- **Side effects:** Calls `free`, releasing heap memory; undefined behavior if `ptr` is invalid or already freed.
- **Calls:** `free`
- **Notes:** Validates that `ptr` is non-NULL before calling `free`. Does not clear the pointer or detect double-free errors.

## Control Flow Notes
These are utility/initialization functions invoked whenever the audio library needs to allocate or deallocate memory (typically during setup/teardown, not per-frame).

## External Dependencies
- `stdlib.h`: `malloc`, `free`
- `usrhooks.h`: Function declarations and error code enum
