# audiolib/public/ps/usrhooks.h

## File Purpose
Public header for the USRHOOKS memory management hook system. Defines the interface contract that allows the audio library to delegate memory allocation and deallocation to the calling program, enabling custom memory management policies (pooling, tracking, restrictions, etc.).

## Core Responsibilities
- Define error codes for hook operations
- Declare memory allocation hook function signature
- Declare memory deallocation hook function signature
- Establish the interface that calling programs must implement to integrate with the audio library

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `USRHOOKS_Errors` | enum | Return codes for hook operations: success (0), error (-1), warning (-2) |

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- **Signature:** `int USRHOOKS_GetMem( void **ptr, unsigned long size )`
- **Purpose:** Allocate memory; called by the audio library when it needs dynamic memory.
- **Inputs:** 
  - `size`: Number of bytes to allocate
  - `ptr`: Output parameter (pointer-to-pointer) to receive the allocated address
- **Outputs/Return:** Error code from `USRHOOKS_Errors` enum (0 = success, -1 = error, -2 = warning)
- **Side effects:** Allocates memory; modifies `*ptr` on success.
- **Calls:** (Not inferable from header—implementation in calling code)
- **Notes:** Calling code must implement this function. The audio library will call it during runtime memory requests.

### USRHOOKS_FreeMem
- **Signature:** `int USRHOOKS_FreeMem( void *ptr )`
- **Purpose:** Deallocate memory; called by the audio library when it no longer needs a block.
- **Inputs:** `ptr`: Address of memory to free
- **Outputs/Return:** Error code from `USRHOOKS_Errors` enum
- **Side effects:** Deallocates memory; must invalidate or track the pointer.
- **Calls:** (Not inferable from header—implementation in calling code)
- **Notes:** Calling code must implement this function. Pairs with USRHOOKS_GetMem.

## Control Flow Notes
These are hook/callback function prototypes. The audio library calls them at runtime whenever memory allocation or freeing is needed. The actual implementations reside in the calling program (usrhooks.c), allowing that program to intercept and control all memory operations of the audio library.

## External Dependencies
- None; this is a pure interface definition with no external includes.
