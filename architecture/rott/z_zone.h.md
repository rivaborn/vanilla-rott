# rott/z_zone.h

## File Purpose
Public interface for Z_Zone, Carmack's memory manager for the engine. Defines memory allocation tags, lifecycle constants, and memory management function declarations. Implements a tagged-memory system where allocations are freed based on lifetime categories (static, game, level, cache).

## Core Responsibilities
- Define memory tag constants (PU_*) that categorize allocations by lifetime
- Declare memory allocation (`Z_Malloc`, `Z_LevelMalloc`) and deallocation (`Z_Free`, `Z_FreeTags`) functions
- Provide heap query and statistics functions (`Z_HeapSize`, `Z_UsedHeap`, `Z_AvailHeap`)
- Expose heap debugging and integrity checking (`Z_DumpHeap`, `Z_CheckHeap`)
- Manage tag reassignment for dynamic purge behavior (`Z_ChangeTag`)

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| lowmemory | int | global | Flag indicating low-memory condition |
| zonememorystarted | int | global | Flag indicating Z_Zone manager initialized |

## Key Functions / Methods

### Z_Init
- **Signature:** `void Z_Init(int size, int min);`
- **Purpose:** Initialize memory manager with heap size and minimum requirement.
- **Inputs:** `size` (heap size in bytes), `min` (minimum required allocation)
- **Outputs/Return:** void
- **Side effects:** Initializes zone memory state; must be called before any allocation
- **Notes:** Foundation call; must precede all other Z_Zone operations

### Z_Malloc
- **Signature:** `void *Z_Malloc(int size, int tag, void *user);`
- **Purpose:** Allocate memory with a purge tag and optional user/callback pointer.
- **Inputs:** `size` (bytes), `tag` (PU_* constant), `user` (optional owner; NULL if tag < PU_PURGELEVEL)
- **Outputs/Return:** Pointer to allocated memory
- **Side effects:** Updates heap state; may trigger purge if purgable tag and memory low

### Z_LevelMalloc
- **Signature:** `void *Z_LevelMalloc(int size, int tag, void *user);`
- **Purpose:** Allocate memory tagged for level scope (PU_LEVEL, PU_LEVELSTRUCT).
- **Inputs:** `size` (bytes), `tag`, `user`
- **Outputs/Return:** Pointer to allocated memory
- **Side effects:** Updates heap; memory persists until level exit

### Z_Free
- **Signature:** `void Z_Free(void *ptr);`
- **Purpose:** Deallocate a single zone-managed pointer.
- **Inputs:** `ptr` (pointer to free)
- **Outputs/Return:** void
- **Side effects:** Marks block free; may enable reuse or purge

### Z_FreeTags
- **Signature:** `void Z_FreeTags(int lowtag, int hightag);`
- **Purpose:** Free all allocations within a tag range.
- **Inputs:** `lowtag`, `hightag` (tag range bounds)
- **Outputs/Return:** void
- **Side effects:** Batch-frees multiple blocks (used for level/game cleanup)

### Z_ChangeTag
- **Signature:** `void Z_ChangeTag(void *ptr, int tag);`
- **Purpose:** Reassign tag of allocated block to change purge eligibility.
- **Inputs:** `ptr`, `tag` (new tag)
- **Outputs/Return:** void
- **Side effects:** Changes lifetime/purge behavior of block

### Z_ShutDown
- **Signature:** `void Z_ShutDown(void);`
- **Purpose:** Shut down memory manager on exit.
- **Outputs/Return:** void
- **Side effects:** Cleans up heap resources

**Other query/debug functions:** `Z_DumpHeap`, `Z_CheckHeap`, `Z_HeapSize`, `Z_UsedHeap`, `Z_AvailHeap`, `Z_UsedStaticHeap`, `Z_UsedLevelHeap`, `Z_GetSize`, `Z_Realloc` — collectively provide heap diagnostics and reallocation.

## Control Flow Notes
Z_Init is called during engine startup. Level allocations use PU_LEVEL (tag 50) or PU_LEVELSTRUCT (tag 49); they are freed en masse via `Z_FreeTags(PU_LEVEL, PU_LEVELEND)` on level exit. Purgable allocations (tags ≥ 100: cache, sprites, sounds, textures) are freed opportunistically when heap pressure requires. Static allocations (tags < 100) persist until explicitly freed or shutdown.

## External Dependencies
None (pure header; no includes or external references).
