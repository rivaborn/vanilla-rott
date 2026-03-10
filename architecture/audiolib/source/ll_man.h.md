# audiolib/source/ll_man.h

## File Purpose
Public header for linked list management routines. Provides generic doubly-linked list operations via type-safe macros and underlying functions, supporting dynamic node insertion/removal and memory locking for audio subsystem data structures.

## Core Responsibilities
- Define error codes for linked list operations
- Provide generic node add/remove functions working with offset-based field access
- Expose convenience macros for type-safe head/tail insertion and removal
- Support memory locking/unlocking during linked list manipulation
- Define minimal list container (start/end pointer pair)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `LL_Errors` | enum | Error/status codes: `LL_Ok`, `LL_Error`, `LL_Warning` |
| `list` | struct | Container holding `void *start` and `void *end` pointers |

## Global / File-Static State
None.

## Key Functions / Methods

### LL_AddNode
- Signature: `void LL_AddNode( char *node, char **head, char **tail, int next, int prev )`
- Purpose: Generic function to insert a node into a doubly-linked list
- Inputs: 
  - `node`: pointer to the node being added (cast to `char*`)
  - `head`, `tail`: pointers to list head/tail pointers
  - `next`, `prev`: byte offsets of `next`/`prev` fields within the node struct
- Outputs/Return: void
- Side effects: Modifies head/tail pointers; updates neighbor node links
- Calls: Not inferable from this file
- Notes: Works with any struct type by computing field offsets at compile-time using `(int)&((type*)0)->fieldname`

### LL_RemoveNode
- Signature: `void LL_RemoveNode( char *node, char **head, char **tail, int next, int prev )`
- Purpose: Generic function to remove a node from a doubly-linked list
- Inputs: 
  - `node`: pointer to node being removed
  - `head`, `tail`: pointers to list head/tail pointers
  - `next`, `prev`: byte offsets of `next`/`prev` fields
- Outputs/Return: void
- Side effects: Updates head/tail pointers and neighbor links; modifies list structure
- Calls: Not inferable from this file
- Notes: Safely updates head/tail if node is at list boundary

### LL_LockMemory
- Signature: `int LL_LockMemory( void )`
- Purpose: Lock memory associated with linked list operations
- Inputs: None
- Outputs/Return: Status code (likely 0 for success)
- Side effects: Prevents memory reallocation or movement
- Calls: Not inferable from this file
- Notes: Paired with `LL_UnlockMemory`; likely for audio/real-time safety

### LL_UnlockMemory
- Signature: `void LL_UnlockMemory( void )`
- Purpose: Unlock previously locked memory
- Inputs: None
- Outputs/Return: void
- Side effects: Allows memory operations to resume
- Calls: Not inferable from this file

## Macros
- `LL_AddToHead()`, `LL_AddToTail()`, `LL_Remove()`: Type-safe wrappers computing field offsets automatically
- `LL_NextNode()`, `LL_PreviousNode()`: Direct pointer dereference helpers

## Control Flow Notes
Infrastructure header used for initialization and runtime list management throughout the audio library. Memory locking likely critical during real-time audio processing to prevent latency from GC/reallocation.

## External Dependencies
- Assumes client code defines structs with `next` and `prev` pointers for intrusive linked list integration
- No external includes visible; self-contained interface
