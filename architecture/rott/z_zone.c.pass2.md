# rott/z_zone.c — Enhanced Analysis

## Architectural Role

This file implements the **foundational memory management subsystem** for the ROTT engine, providing allocation/deallocation with automatic prioritization-based purging. It sits at a critical architectural layer—nearly all game resources (actors, textures, level data, network buffers) flow through this allocator. The two-zone design (main + level) reflects the game's separation between persistent game state and per-level ephemeral resources; the level zone size is dynamically tuned at startup based on network multiplayer configuration, coupling memory management tightly to the networking subsystem.

## Key Cross-References

### Incoming (who depends on this file)
- **Virtually all subsystems**: Game actor system (`rt_actor.c`), player system (`rt_playr.c`), static objects (`rt_stat.c`), level loader, cinematic system (`cin_*.c`), networking (`rt_net.c`) all allocate via `Z_Malloc()` / `Z_LevelMalloc()`
- **Game initialization**: `Z_Init()` called early in `rt_main.c` (or equivalent startup code) with calculated memory budget
- **Level transitions**: `Z_FreeTags()` called by level-exit code to flush tag ranges `PU_LEVEL` through `PU_LEVELEND`
- **Shutdown path**: `Z_ShutDown()` called at game exit
- **Debug/profiling**: `Z_DumpHeap()`, `Z_CheckHeap()` called from debug menu or periodic assertions (`rt_debug.c`)

### Outgoing (what this file depends on)
- **Networking/configuration**: Reads `GamePacketSize()`, `ConsoleIsServer()` to calculate level zone size (server needs 2× client buffer overhead per player)
- **Error handling**: Calls `Error()` (fatal), `SoftError()` (recoverable logging), `UL_DisplayMemoryError()` (UI dialog)
- **System/DPMI**: Calls `int386x()` to query available contiguous memory via DOS protected-mode interface (fills `MemInfo` struct)
- **Utility**: Calls `CheckParm()` from `rt_util.c` (likely for command-line debug flags); uses `SafeMalloc()`/`SafeFree()` in `Z_Realloc()` (not zone-based—potential design inconsistency)
- **I/O**: Calls `printf()`, `getch()` for console feedback and user pauses (DOS-era blocking UI)

## Design Patterns & Rationale

### Rover-Based First-Fit Allocation
- **Pattern**: The `rover` pointer tracks the last successful allocation; next search starts nearby rather than from zone head
- **Rationale**: Most allocations are sequential (level load → actor spawn → frame-by-frame transients), so rover minimizes traversal. Reduces O(n) worst-case for typical workloads
- **Tradeoff**: External fragmentation possible if allocations are randomly-sized; coalescing only merges *adjacent* free blocks

### Tag-Based Prioritization (Purgeability)
- **Pattern**: Each block tagged with `tag` (0–255); blocks with `tag ≥ PU_PURGELEVEL` (≈100) are purgeable caches
- **Rationale**: Simple scheme avoiding reference counting or GC pause concerns. Game can pre-allocate high-priority cache and let low-priority ephemera be evicted
- **Tradeoff**: No granular priorities within purgeable tier; all purgeable blocks treated equally (FIFO-like eviction by rover scan order)

### Dual-Zone Separation
- **Pattern**: `mainzone` for general game data; `levelzone` for level-specific resources
- **Rationale**: Allows bulk deallocation (`Z_FreeTags()` on level tags) without scanning entire heap; also isolates level memory fragmentation from persistent state
- **Tradeoff**: Requires caller to choose allocator (`Z_Malloc` vs `Z_LevelMalloc`); no automatic migration between zones

### User-Pointer Update Mechanism
- **Pattern**: Caller optionally passes `user` pointer (address of a variable); allocator writes back the allocated address
- **Rationale**: Simplifies deallocation—pointer auto-invalidated on purge (set to `NULL`), so caller can check validity without external tracking
- **Tradeoff**: Requires double-pointer; caller must initialize to `NULL`

## Data Flow Through This File

```
Startup:
  Z_Init(maxsize, minsize)
    ├─ Calculate packet overhead from GamePacketSize() + ConsoleIsServer()
    ├─ Allocate mainzone, levelzone via malloc()
    └─ Z_ClearZone() → each zone = single free block, rover = start

Level Load:
  Z_LevelMalloc(..., tag=PU_LEVEL, user=&ptr)
    ├─ Scan from levelzone->rover for first fit
    ├─ Purge if needed: Z_Free() on purgeable blocks
    └─ Write &ptr = allocated_addr; mark block with PU_LEVEL tag

Game Loop (transients):
  Z_Malloc(..., tag=PU_CACHE, user=NULL)
    ├─ Scan from mainzone->rover
    ├─ Auto-purge purgeable blocks on demand
    └─ Return unowned cache block

Level Exit:
  Z_FreeTags(PU_LEVEL, PU_LEVELEND)
    ├─ Iterate both zones
    ├─ Z_Free() all blocks in tag range
    ├─ Adjacent free blocks coalesce immediately
    └─ Next Z_LevelMalloc has clean slate

Shutdown:
  Z_ShutDown()
    └─ free(mainzone); free(levelzone)
```

## Learning Notes

### DOS-Era Memory Management
- **DPMI (DOS Protected Mode Interface)**: `Z_AvailHeap()` queries largest contiguous block via interrupt 31h (int386x), reflecting 386+ protected mode but still DOS-compatible. Modern engines query OS virtual memory directly.
- **Far pointers**: Uses `FP_SEG()`, `FP_OFF()` macros to decompose pointers into segment:offset (real-mode legacy artifact preserved in protected mode for compatibility).
- **Memory models**: Assumes flat address space post-286; pre-allocates from malloc, not dynamic growth.

### Idiomatic Design for 1990s Game Engines
- **Tag-based purging** is simpler than modern approaches (weak references, generational GC) and avoids stop-the-world pauses—critical for real-time games.
- **Two-tier zones** prefigure modern layer-based architectures (persistent vs. frame-local allocators), though without the sophistication of modern arena or linear allocators.
- **Rover optimization** is a pragmatic heuristic before true working-set analysis or allocation profiling became standard.

### Comparison to Modern Approaches
| Aspect | z_zone.c | Modern Engines |
|--------|----------|---|
| Fragmentation | Linked-list coalescing | Arena/bump allocators, defrag passes |
| Purging | Tag-based FIFO scan | Ref-count / weak references / GC |
| Query | DPMI interrupt | OS API (VirtualAlloc, mmap) |
| Multi-zone | Manual (mainzone/levelzone) | Implicit via arena type |
| Corruption detection | Pre/post tags (compile-time) | Valgrind, ASAN runtime checks |

## Potential Issues

1. **Z_Realloc design inconsistency** (line ~760): Uses `SafeMalloc()`/`SafeFree()` from elsewhere, not the zone allocator. If `SafeMalloc()` calls malloc directly, reallocated pointers escape zone tracking and cannot be purged or bulk-freed. Should use `Z_Malloc()` instead.

2. **Error handling in Z_Malloc/Z_LevelMalloc**: Loop can theoretically deadlock if all purgeable blocks are purged but no free block exists (condition `base->user || base->size < size` remains true). However, `SoftError()` and `Z_DumpHeap()` called first, suggesting developer awareness of this edge case.

3. **No accounting for DPMI query overhead**: `Z_AvailHeap()` returns DPMI-reported contiguous block but does not account for internal zone fragmentation. A caller might see "8 MB available" from DPMI but fail Z_Malloc due to fragmentation into 100 KB chunks.

4. **Tag-range freeing is not selective**: `Z_FreeTags(PU_LEVEL, PU_LEVELEND)` frees *all* blocks in range from both zones, indiscriminately. If a block is tagged PU_LEVEL but logically should persist, it will be freed. No protection against accidental tagging.

5. **Rover can point to unowned cache blocks**: After purging a cache block that was the rover, rover repositions to adjacent block. If that adjacent block is also purgeable and immediately eligible for eviction, next allocation might again trigger purge—pathological case if cache is near-empty.
