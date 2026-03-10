# audiolib/public/include/usrhooks.h

## File Purpose
Public header for user hook interface in the audio library. Defines memory management function prototypes that allow the calling program to provide custom allocators and deallocators, enabling the audio library to respect application-specific memory constraints.

## Core Responsibilities
- Define memory allocation hook (`USRHOOKS_GetMem`)
- Define memory deallocation hook (`USRHOOKS_FreeMem`)
- Define error codes for hook operation results

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `USRHOOKS_Errors` | enum | Return status codes for hook functions |

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- Signature: `int USRHOOKS_GetMem( void **ptr, unsigned long size )`
- Purpose: Allocate memory on behalf of the audio library
- Inputs: `size` (bytes to allocate); `ptr` (output pointer to receive allocation)
- Outputs/Return: `USRHOOKS_Errors` enum value (`USRHOOKS_Ok` on success, `USRHOOKS_Error` on failure)
- Side effects: Allocates memory; writes result to `*ptr`
- Calls: None (declaration only)
- Notes: Double-indirection (`void **ptr`) allows in-place initialization of caller's pointer

### USRHOOKS_FreeMem
- Signature: `int USRHOOKS_FreeMem( void *ptr )`
- Purpose: Free memory previously allocated via `USRHOOKS_GetMem`
- Inputs: `ptr` (pointer to deallocate)
- Outputs/Return: `USRHOOKS_Errors` enum value
- Side effects: Deallocates memory
- Calls: None (declaration only)
- Notes: Caller responsible for ensuring pointer validity

## Control Flow Notes
This header defines a **plugin interface**. Actual implementations live in USRHOOKS.C (implementation) and are provided by the calling application. The audio library calls these hooks whenever memory allocation/deallocation is needed, delegating control to the application.

## External Dependencies
- Standard C (no external includes in this header)
- Implementations defined elsewhere (`USRHOOKS.C`)
