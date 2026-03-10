# audiolib/public/pm/source/usrhooks.h

## File Purpose
Public header file defining custom memory management hook functions for the audio library. Provides an abstraction layer allowing calling programs to override or intercept memory allocation and deallocation operations performed by the library.

## Core Responsibilities
- Define status/error codes for memory hook operations
- Declare memory allocation hook function prototype
- Declare memory deallocation hook function prototype
- Enable application-level control of library memory management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| USRHOOKS_Errors | enum | Status codes returned by hook functions: Ok (0), Error (-1), Warning (-2) |

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- Signature: `int USRHOOKS_GetMem(void **ptr, unsigned long size)`
- Purpose: Allocate memory; allows calling program to provide custom allocation strategy
- Inputs: `size` (bytes to allocate)
- Outputs/Return: Populates `ptr` with allocated memory address; returns status code (USRHOOKS_Ok, USRHOOKS_Error, or USRHOOKS_Warning)
- Side effects: Memory allocation; pointer modification
- Calls: Implemented elsewhere (not in this header)
- Notes: Uses double pointer (`void **`) to return allocated address; caller responsible for pointer validity

### USRHOOKS_FreeMem
- Signature: `int USRHOOKS_FreeMem(void *ptr)`
- Purpose: Deallocate memory; allows calling program to provide custom deallocation strategy
- Inputs: `ptr` (memory address to free)
- Outputs/Return: Status code (USRHOOKS_Ok, USRHOOKS_Error, or USRHOOKS_Warning)
- Side effects: Memory deallocation
- Calls: Implemented elsewhere (not in this header)
- Notes: Assumes valid pointer; no validation in header

## Control Flow Notes
These are hook/callback functions that the audio library calls instead of direct malloc/free. The calling application implements these functions to control memory allocation behavior (e.g., using custom allocators, memory pools, or tracking).

## External Dependencies
None. Self-contained header with no external includes or dependencies.
