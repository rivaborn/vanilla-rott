# audiolib/source/ll_man.c

## File Purpose
Implements generic doubly-linked list management for an audio library. Provides functions to insert and remove nodes from lists using offset-based pointer arithmetic, with optional memory locking for DOS real-mode environments.

## Core Responsibilities
- Insert nodes at the head of a doubly-linked list
- Remove nodes from a doubly-linked list
- Lock critical linked-list functions in memory (DOS real-mode requirement)
- Unlock memory regions when no longer needed
- Support type-generic linked-list operations via macro-based offset calculations

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| LOCKMEMORY | Macro flag | Compile-time | Enables DOS DPMI memory locking when defined |
| OFFSET | Macro | File-static | Pointer arithmetic helper; converts byte offset into a dereferenced pointer |
| LL_LockStart | Function pointer alias | File-static | References LL_AddNode; marks start of memory-lockable region |
| LL_LockEnd | Function | File-static | Sentinel function marking end of memory-lockable region |

## Key Functions / Methods

### LL_AddNode
- **Signature:** `void LL_AddNode(char *item, char **head, char **tail, int next, int prev)`
- **Purpose:** Insert a node at the head of a doubly-linked list.
- **Inputs:**
  - `item`: pointer to node to insert (generic; cast to char*)
  - `head`: pointer to list head pointer
  - `tail`: pointer to list tail pointer
  - `next`: byte offset of "next" field within node structure
  - `prev`: byte offset of "prev" field within node structure
- **Outputs/Return:** None (void).
- **Side effects:** Modifies `*head` to point to new node; updates `*tail` if list was empty; updates prev/next pointers via OFFSET macro.
- **Calls:** OFFSET macro (pointer arithmetic).
- **Notes:** Assumes node has `next` and `prev` fields at specified offsets. If list is empty, initializes tail. Works for any type via offset-based access.

### LL_RemoveNode
- **Signature:** `void LL_RemoveNode(char *item, char **head, char **tail, int next, int prev)`
- **Purpose:** Remove a node from a doubly-linked list.
- **Inputs:**
  - `item`: pointer to node to remove
  - `head`: pointer to list head pointer
  - `tail`: pointer to list tail pointer
  - `next`: byte offset of "next" field within node
  - `prev`: byte offset of "prev" field within node
- **Outputs/Return:** None (void).
- **Side effects:** Updates `*head` if removing the head node; updates `*tail` if removing the tail node; clears the removed node's next/prev pointers.
- **Calls:** OFFSET macro.
- **Notes:** Handles edge cases: removing head, tail, and middle nodes. Clears node pointers to prevent stale references.

### LL_LockMemory
- **Signature:** `int LL_LockMemory(void)`
- **Purpose:** Lock the linked-list functions in physical memory (DOS real-mode requirement to prevent page faults).
- **Inputs:** None.
- **Outputs/Return:** `LL_Ok` on success; `LL_Error` if DPMI locking fails.
- **Side effects:** Calls DPMI_LockMemoryRegion if LOCKMEMORY is defined; may fail on systems without DPMI support.
- **Calls:** DPMI_LockMemoryRegion (defined elsewhere).
- **Notes:** Only active when LOCKMEMORY is defined. Uses LL_LockStart and LL_LockEnd to determine region boundaries.

### LL_UnlockMemory
- **Signature:** `void LL_UnlockMemory(void)`
- **Purpose:** Unlock the linked-list functions from physical memory.
- **Inputs:** None.
- **Outputs/Return:** None (void).
- **Side effects:** Calls DPMI_UnlockMemoryRegion if LOCKMEMORY is defined.
- **Calls:** DPMI_UnlockMemoryRegion (defined elsewhere).
- **Notes:** Pairs with LL_LockMemory. Only active when LOCKMEMORY is defined.

## Control Flow Notes
This module supports initialization and shutdown phases:
- **Initialization:** LL_LockMemory() locks critical code at engine startup to prevent real-mode page faults.
- **Runtime:** LL_AddNode/LL_RemoveNode are called whenever audio lists (likely playback queues or sound object lists) are modified.
- **Shutdown:** LL_UnlockMemory() releases memory locks.

## External Dependencies
- `<stddef.h>`: Standard C definitions (size_t, NULL).
- **ll_man.h**: Declares public API and macros (LL_AddToHead, LL_AddToTail, LL_Remove).
- **dpmi.h**: DOS Protected Mode Interface declarations for memory locking (DPMI_LockMemoryRegion, DPMI_UnlockMemoryRegion — defined elsewhere).
