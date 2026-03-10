# rott/usrhooks.h

## File Purpose
Public header file defining the memory management hook interface for the ROTT engine. Provides abstraction for memory allocation and deallocation operations that may be restricted or customized by the calling program.

## Core Responsibilities
- Define error codes for user hook operations
- Declare memory allocation hook function prototype
- Declare memory deallocation hook function prototype
- Enable libraries to interface with caller-controlled memory management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| USRHOOKS_Errors | enum | Error codes for hook operations (Warning, Error, Ok) |

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- **Signature:** `int USRHOOKS_GetMem( void **ptr, unsigned long size )`
- **Purpose:** Allocate memory through a caller-controlled hook
- **Inputs:** `ptr` (output pointer to allocated memory), `size` (bytes to allocate)
- **Outputs/Return:** Integer status code (USRHOOKS_Ok, USRHOOKS_Error, or USRHOOKS_Warning)
- **Side effects:** Allocates memory; modifies `*ptr` to point to allocated block
- **Calls:** (declaration only; implementation elsewhere)
- **Notes:** Caller is responsible for implementing the actual allocation strategy

### USRHOOKS_FreeMem
- **Signature:** `int USRHOOKS_FreeMem( void *ptr )`
- **Purpose:** Deallocate memory through a caller-controlled hook
- **Inputs:** `ptr` (pointer to memory block to free)
- **Outputs/Return:** Integer status code (USRHOOKS_Ok, USRHOOKS_Error, or USRHOOKS_Warning)
- **Side effects:** Deallocates memory
- **Calls:** (declaration only; implementation elsewhere)
- **Notes:** Caller is responsible for implementing the actual deallocation strategy

## Control Flow Notes
This header defines a plugin interface for memory operations. The actual hook implementations are expected to be provided in `USRHOOKS.C` (referenced in the comment). Libraries and engine components call these functions instead of using standard `malloc`/`free`, allowing the host program to control memory allocation behavior (e.g., custom allocators, memory pools, restricted heaps).

## External Dependencies
- Standard C (no external dependencies; declarations only)
- Implementation defined in `USRHOOKS.C` (mentioned in module header)
