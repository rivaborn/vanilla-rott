# audiolib/source/linklist.h

## File Purpose
Provides a collection of macro-based utilities for managing doubly-linked circular lists in C. This generic, type-agnostic approach avoids code duplication by parameterizing node types and pointer field names.

## Core Responsibilities
- Create and initialize empty linked lists
- Add, remove, and move nodes within lists
- Transfer entire lists between roots
- Reverse list node ordering
- Insert nodes in sorted order
- Check list state (empty/non-empty)
- Deallocate list memory

## Key Types / Data Structures
None. Macros are generic and assume node types with `next` and `prev` pointer fields (struct layout defined by caller).

## Global / File-Static State
None.

## Key Functions / Methods

### LL_CreateNewLinkedList
- Signature: `LL_CreateNewLinkedList(rootnode, type, next, prev)`
- Purpose: Initialize a new empty circular doubly-linked list
- Inputs: `rootnode` (ptr to root), `type` (node struct type), `next`/`prev` (pointer field names)
- Outputs: `rootnode` initialized with circular self-references
- Side effects: Calls `SafeMalloc(sizeof(type))`
- Calls: `NewNode`
- Notes: Creates circular structure where root points to itself; starting point for all list operations

### LL_AddNode
- Signature: `LL_AddNode(rootnode, newnode, next, prev)`
- Purpose: Insert node into list (appends before root in circular structure)
- Inputs: `rootnode`, `newnode` (node to insert), field names
- Outputs: `newnode` linked into chain
- Side effects: Modifies 4 pointer assignments
- Notes: Assumes `newnode` is unlinked; O(1) insertion

### LL_RemoveNode
- Signature: `LL_RemoveNode(node, next, prev)`
- Purpose: Unlink node from its list
- Inputs: `node` to remove, field names
- Outputs: `node` isolated (next/prev point to itself)
- Side effects: Pointer chain update
- Notes: After removal, node is in "empty list" state

### LL_SortedInsertion
- Signature: `LL_SortedInsertion(rootnode, insertnode, next, prev, type, sortparm)`
- Purpose: Insert node in sorted position based on a comparable field
- Inputs: `rootnode`, `insertnode`, field names, `type`, `sortparm` (member to sort by)
- Outputs: `insertnode` placed in order
- Side effects: O(n) linear scan + insertion
- Calls: `LL_AddNode`
- Notes: Assumes `sortparm` is numeric/comparable; descending order

### LL_MoveNode
- Signature: `LL_MoveNode(node, newroot, next, prev)`
- Purpose: Remove node from current list, add to `newroot`
- Outputs: `node` transferred atomically
- Calls: `LL_RemoveNode`, `LL_AddNode`

### LL_TransferList
- Signature: `LL_TransferList(oldroot, newroot, next, prev)`
- Purpose: Move all nodes from `oldroot` to `newroot` (merges lists)
- Outputs: `oldroot` becomes empty (circular self-reference)
- Notes: Only executes if `oldroot` non-empty

### LL_ReverseList
- Signature: `LL_ReverseList(root, type, next, prev)`
- Purpose: Reverse node ordering in list
- Side effects: Modifies all pointers in chain
- Calls: `LL_MoveNode`

### LL_ListEmpty
- Signature: `LL_ListEmpty(list, next, prev)` → boolean
- Purpose: Test if list contains nodes
- Notes: True iff root points to itself

## Control Flow Notes
Utility header with no inherent control flow. Macros are inlined at call sites throughout the audio library for list manipulation.

## External Dependencies
- `SafeMalloc`, `SafeFree` (memory management—defined elsewhere)
- C++ compatible via `extern "C"` wrapper
