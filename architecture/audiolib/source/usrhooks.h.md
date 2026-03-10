# audiolib/source/usrhooks.h

## File Purpose
Public header defining memory allocation hooks for the audio library. Allows the calling program to control memory management operations that the audio library performs, enabling custom allocation strategies or restricted environments.

## Core Responsibilities
- Define error codes for hook function operations
- Declare function prototypes for memory allocation/deallocation
- Establish a standard interface between audio library and caller for memory operations
- Enable the calling program to intercept and customize memory management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| USRHOOKS_Errors | enum | Error codes returned by hook functions |

## Global / File-Static State
None.

## Key Functions / Methods

### USRHOOKS_GetMem
- Signature: `int USRHOOKS_GetMem( void **ptr, unsigned long size )`
- Purpose: Allocate a memory block of specified size; called by audio library when it needs memory
- Inputs: `size` (number of bytes to allocate)
- Outputs/Return: `ptr` (pointer to allocated block, output via double-pointer); return code (USRHOOKS_Ok on success)
- Side effects: Allocates memory from caller's heap; may fail if memory unavailable
- Notes: Called by the audio library; caller must implement this function with desired allocation strategy

### USRHOOKS_FreeMem
- Signature: `int USRHOOKS_FreeMem( void *ptr )`
- Purpose: Deallocate a memory block previously allocated by USRHOOKS_GetMem
- Inputs: `ptr` (pointer to block to free)
- Outputs/Return: Return code (USRHOOKS_Ok on success)
- Side effects: Deallocates memory; must match allocation performed by USRHOOKS_GetMem
- Notes: Called by audio library; caller must implement to match their allocation scheme

## Control Flow Notes
Not inferable from this file. These are hook functions—the audio library calls them during its own initialization and cleanup. The calling program provides implementations.

## External Dependencies
None.
