# audiolib/public/timer/source/usrhooks.h

## File Purpose
Public header declaring memory management hook functions for the USRHOOKS module. Provides a callback interface that allows the calling program to intercept or customize memory allocation and deallocation operations required by the audio library.

## Core Responsibilities
- Define error codes for hook operations
- Declare allocation hook function (USRHOOKS_GetMem)
- Declare deallocation hook function (USRHOOKS_FreeMem)
- Establish a contract for custom memory management in the audio library

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| USRHOOKS_Errors | enum | Error/status codes returned by hook functions |

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- Signature: `int USRHOOKS_GetMem(void **ptr, unsigned long size)`
- Purpose: Memory allocation hook; allows the caller to implement custom allocation logic
- Inputs: `ptr` (pointer to pointer for output), `size` (bytes to allocate)
- Outputs/Return: Status code (USRHOOKS_Ok on success, USRHOOKS_Error/-1 on failure)
- Side effects: Must allocate memory and write address to `*ptr`; caller-defined behavior
- Calls: Not visible (implementation is caller-provided)
- Notes: Double pointer pattern allows the hook to return allocated address; size is unsigned long (supports large allocations)

### USRHOOKS_FreeMem
- Signature: `int USRHOOKS_FreeMem(void *ptr)`
- Purpose: Memory deallocation hook; allows the caller to implement custom freeing logic
- Inputs: `ptr` (address of memory to free)
- Outputs/Return: Status code (USRHOOKS_Ok on success, USRHOOKS_Error/-1 on failure)
- Side effects: Must deallocate memory at `ptr`; caller-defined behavior
- Calls: Not visible (implementation is caller-provided)
- Notes: Complements USRHOOKS_GetMem; error codes allow graceful failure handling

## Control Flow Notes
These are hook/callback functions that the audio library calls when it needs memory operations. The calling application must implement these functions to control how the library allocates and frees memory. This pattern decouples the library from direct malloc/free calls, enabling custom memory management (pooling, tracking, restricted heaps, etc.).

## External Dependencies
- Standard C (no explicit includes in this header)
- Caller must implement the two declared functions
