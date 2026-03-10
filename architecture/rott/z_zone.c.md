# rott/z_zone.c

## File Purpose
Implements a custom zone-based memory allocator for the ROTT game engine, supporting two separate zones (main and level) with automatic fragmentation management. Provides memory allocation with tagging-based purging levels, allowing lower-priority blocks to be freed when allocation pressure requires.

## Core Responsibilities
- Allocate and deallocate memory blocks from main and level zones
- Track allocated blocks with metadata (size, owner pointer, purge tag)
- Automatically purge purgeable blocks (tag ≥ 100) when allocation fails
- Coalesce adjacent free blocks to reduce fragmentation
- Maintain memory statistics and heap validation for debugging
- Query available contiguous memory via DPMI interrupt
- Support level transitions by bulk-freeing memory in tag ranges

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `memblock_t` | struct | Single memory block header; tracks size, user pointer, tag, linked-list pointers, and optional pre/post corruption tags |
| `memzone_t` | struct | Zone structure containing total size, doubly-linked block list, and rover (allocation search pointer) |
| `struct meminfo` | struct | DPMI memory info returned by int386x; contains available contiguous block size and other memory statistics |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `lowmemory` | int | global | Flag indicating system is running with insufficient memory |
| `zonememorystarted` | int | global | Flag preventing re-initialization of zones |
| `mainzone` | memzone_t* | static | Pointer to primary memory zone for general allocations |
| `levelzone` | memzone_t* | static | Pointer to level-specific memory zone |
| `levelzonesize` | int | static | Size of level zone, adjusted at init based on multiplayer packet size |
| `MemInfo` | struct meminfo | static | DPMI memory information structure |

## Key Functions / Methods

### Z_Init
- **Signature:** `void Z_Init(int size, int min)`
- **Purpose:** Initialize memory zones at startup, adjusting level zone size based on network packet requirements and multiplayer count
- **Inputs:** `size` – requested main zone size; `min` – minimum acceptable size
- **Outputs/Return:** None; calls `Error()` if insufficient memory available
- **Side effects:** Allocates mainzone and levelzone via malloc; sets `zonememorystarted=1`; prints warning and calls `getch()` if low memory detected
- **Calls:** `Z_AllocateZone()`, `Z_ClearZone()`, `GamePacketSize()`, `ConsoleIsServer()`, `UL_DisplayMemoryError()`, `printf()`
- **Notes:** Level zone size increased by network packet multiplier; respects `MAXMEMORYSIZE` cap; issues user warning if available memory < 1.5× minimum

### Z_AllocateZone
- **Signature:** `memzone_t *Z_AllocateZone(int size)`
- **Purpose:** Allocate and initialize a single memory zone structure
- **Inputs:** `size` – bytes to allocate (excluding zone header)
- **Outputs/Return:** Pointer to initialized `memzone_t` structure
- **Side effects:** Calls `malloc()` for zone + header; calls `Z_ClearZone()` to initialize
- **Calls:** `malloc()`, `Z_ClearZone()`, `Error()`, `Z_AvailHeap()`
- **Notes:** Fails with error message if malloc returns NULL

### Z_ClearZone
- **Signature:** `void Z_ClearZone(memzone_t *zone)`
- **Purpose:** Initialize a zone to contain a single free block spanning entire zone size
- **Inputs:** `zone` – zone structure to clear
- **Outputs/Return:** None
- **Side effects:** Modifies zone's blocklist and rover
- **Calls:** None
- **Notes:** Creates sentinel blocklist entry; rover points to initial free block; used after zone allocation and during testing

### Z_Malloc
- **Signature:** `void *Z_Malloc(int size, int tag, void *user)`
- **Purpose:** Allocate memory from main zone with automatic purgeable-block freeing
- **Inputs:** `size` – bytes needed; `tag` – priority tag (< 100 = non-purgeable, ≥ 100 = purgeable); `user` – optional double-pointer to update with block address
- **Outputs/Return:** Pointer to allocated memory (offset past block header)
- **Side effects:** Updates rover; may call `Z_Free()` on purgeable blocks; writes to user pointer if provided; adds pre/post corruption tags if `MEMORYCORRUPTIONTEST=1`
- **Calls:** `Z_Free()`, `SoftError()`, `Z_DumpHeap()`, `Error()`
- **Notes:** Rover-based first-fit algorithm; coalesces fragments > MINFRAGMENT bytes; errors if tag ≥ 100 without user pointer; marks unowned purgeable blocks as `(void*)2`

### Z_LevelMalloc
- **Signature:** `void *Z_LevelMalloc(int size, int tag, void *user)`
- **Purpose:** Allocate memory from level zone (semantically equivalent to Z_Malloc but uses separate zone)
- **Inputs:** Same as Z_Malloc
- **Outputs/Return:** Same as Z_Malloc
- **Side effects:** Updates levelzone rover; calls Z_Free on levelzone blocks
- **Calls:** `Z_Free()`, `SoftError()`, `Z_DumpHeap()`, `Error()`
- **Notes:** Identical algorithm to Z_Malloc; allows independent management of level vs. main memory lifecycles

### Z_Free
- **Signature:** `void Z_Free(void *ptr)`
- **Purpose:** Free a memory block and coalesce with adjacent free blocks
- **Inputs:** `ptr` – pointer returned by Z_Malloc/Z_LevelMalloc
- **Outputs/Return:** None
- **Side effects:** Marks block as free; updates user pointer if valid (> 0x100); merges with previous/next free blocks; updates rover if affected block was rover
- **Calls:** None
- **Notes:** Errors if block already free; coalesces forward and backward; no limit on maximum fragment size; operates on both zones transparently

### Z_FreeTags
- **Signature:** `void Z_FreeTags(int lowtag, int hightag)`
- **Purpose:** Free all allocated blocks with tags within a range (used for level transitions)
- **Inputs:** `lowtag`, `hightag` – inclusive tag range
- **Outputs/Return:** None
- **Side effects:** Iterates both mainzone and levelzone; calls Z_Free on matching blocks
- **Calls:** `Z_Free()`
- **Notes:** Skips already-free blocks; used to flush level-specific memory (`PU_LEVEL` to `PU_LEVELEND`)

### Z_DumpHeap
- **Signature:** `void Z_DumpHeap(int lowtag, int hightag)`
- **Purpose:** Debug output of heap structure for diagnosing fragmentation and allocation issues
- **Inputs:** `lowtag`, `hightag` – tag range filter for display
- **Outputs/Return:** None (uses SoftError for output)
- **Side effects:** Iterates entire block lists in both zones; prints block details and totals; validates structure integrity
- **Calls:** `SoftError()`
- **Notes:** Outputs block pointers, sizes, user pointers, tags; checks for corruption (size touching next, back links, consecutive free blocks)

### Z_CheckHeap
- **Signature:** `void Z_CheckHeap(void)`
- **Purpose:** Validate heap integrity for corruption detection
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Iterates both zones; calls Error() on corruption
- **Calls:** `Error()`
- **Notes:** Checks block size touching, back-link validity, no consecutive free blocks; validates pre/post corruption tags if `MEMORYCORRUPTIONTEST=1`; called periodically during development

### Z_ChangeTag
- **Signature:** `void Z_ChangeTag(void *ptr, int tag)`
- **Purpose:** Change purge priority of an allocated block
- **Inputs:** `ptr` – allocated pointer; `tag` – new priority tag
- **Outputs/Return:** None
- **Side effects:** Modifies block metadata
- **Calls:** None
- **Notes:** Simple tag update; does not reorder blocks

### Z_AvailHeap
- **Signature:** `int Z_AvailHeap(void)`
- **Purpose:** Query largest contiguous free block size via DPMI
- **Inputs:** None
- **Outputs/Return:** Largest available contiguous memory block in bytes
- **Side effects:** Executes DPMI interrupt (int386x); uses FP_SEG/FP_OFF macros
- **Calls:** `int386x()`, `memset()`
- **Notes:** Uses DOS protected-mode interface; only returns DPMI-reported size, not internal zone fragmentation

### Z_UsedHeap / Z_UsedLevelHeap / Z_UsedStaticHeap
- **Signature:** `int Z_UsedHeap(void)`, `int Z_UsedLevelHeap(void)`, `int Z_UsedStaticHeap(void)`
- **Purpose:** Return total bytes in use (main zone, level zone, and static-tagged blocks respectively)
- **Inputs:** None
- **Outputs/Return:** Bytes in use
- **Side effects:** None (read-only iteration)
- **Calls:** None
- **Notes:** Z_UsedStaticHeap filters for tag < PU_PURGELEVEL; simple linear scans

### Z_HeapSize
- **Signature:** `int Z_HeapSize(void)`
- **Purpose:** Get total main zone size (allocated + free)
- **Inputs:** None
- **Outputs/Return:** Total zone size in bytes
- **Side effects:** None
- **Calls:** None

### Z_GetSize
- **Signature:** `int Z_GetSize(void *ptr)`
- **Purpose:** Retrieve size of an allocated block
- **Inputs:** `ptr` – allocated pointer
- **Outputs/Return:** Block size minus header size
- **Side effects:** None
- **Calls:** None

### Z_ShutDown
- **Signature:** `void Z_ShutDown(void)`
- **Purpose:** Deallocate both zones and mark system shut down
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `free()` on mainzone and levelzone; sets `zonememorystarted=0`
- **Calls:** `free()`
- **Notes:** Idempotent (guards against double-free with `zonememorystarted` check)

### Z_Realloc
- **Signature:** `void Z_Realloc(void **ptr, int newsize)`
- **Purpose:** Reallocate block to new size (similar to C realloc but updates double-pointer)
- **Inputs:** `ptr` – address of pointer variable; `newsize` – new size in bytes
- **Outputs/Return:** None; modifies `*ptr` in place
- **Side effects:** Allocates new block, copies data, frees old block
- **Calls:** `SafeMalloc()`, `memcpy()`, `SafeFree()`
- **Notes:** Implemented as allocate-copy-free; not optimal if realloc shrinks (copies unnecessary bytes); does not use zone allocator

## Control Flow Notes
- **Initialization phase:** `Z_Init()` called at game startup; allocates and clears both zones; may display memory warning
- **Level load phase:** `Z_LevelMalloc()` used for level-specific resources
- **Main game loop:** `Z_Malloc()` used for transient allocations; purgeable blocks freed on demand
- **Level exit phase:** `Z_FreeTags()` called with level tag range to flush level memory
- **Shutdown phase:** `Z_ShutDown()` called at game exit to release zones
- **Debugging:** `Z_DumpHeap()` and `Z_CheckHeap()` called on-demand or periodically in development builds

## External Dependencies
- **System headers:** `<stdio.h>`, `<stdlib.h>`, `<dos.h>`, `<string.h>`, `<conio.h>` – DOS/C runtime
- **Internal headers:** `rt_def.h` (constants), `_z_zone.h` (private structures), `z_zone.h` (public interface), `rt_util.h` (Error, SoftError, SafeMalloc, SafeFree, CheckParm)
- **Conditional headers:** `rt_main.h` (if DEVELOPMENT=1), `develop.h` (feature flags), `memcheck.h` (unused)
- **External functions used:** `GamePacketSize()`, `ConsoleIsServer()`, `UL_DisplayMemoryError()`, `int386x()` (DPMI), `Error()`, `SoftError()`, `SafeMalloc()`, `SafeFree()` – defined elsewhere
- **Macros/constants:** `PU_*` tags from z_zone.h; `MINFRAGMENT`, `MAXMEMORYSIZE`, `LEVELZONESIZE` from _z_zone.h; `FP_SEG()`, `FP_OFF()` for far pointers
