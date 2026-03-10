# rott/_z_zone.h — Enhanced Analysis

## Architectural Role

This header defines the **foundational data structures for ROTT's zone-based memory management subsystem**, which implements a custom heap allocator for dynamic memory throughout the engine's lifetime. The zone allocator partitions memory into discrete pools (zones) with independent lifecycles—critical for a game engine managing level-specific memory, dynamic entities, and real-time constraints. The `memblock_t` and `memzone_t` structures form the core of a first-fit allocation strategy optimized for 1990s-era DOS/protected-mode systems where memory fragmentation and corruption detection were severe concerns.

## Key Cross-References

### Incoming (who depends on this file)
- **Zone allocation functions** (`Z_Malloc`, `Z_Free`, `Z_Purge`, etc., defined in companion `.c` files) are called throughout the engine for dynamic memory: actor/enemy spawning, sprite caching, level geometry, texture/sprite data, and save-state buffers
- The `memzone_t` structure (and its memory pool) is initialized during engine startup and referenced in core loops (frame updates, rendering, actor movement)
- Memory corruption detection (`MEMORYCORRUPTIONTEST`) is a compile-time feature for debug builds—enabled when developer builds need to catch heap-stomping bugs

### Outgoing (what this file depends on)
- Includes `develop.h` for compile-time feature flags controlling debug instrumentation
- The `DPMI_INT` constant suggests integration with DOS memory management and protected-mode memory allocation (likely called by zone initialization code)

## Design Patterns & Rationale

**Pool Allocation with Rover Optimization**: The `memzone_t.rover` pointer implements a first-fit allocator variant. Instead of always searching from the head of the block list, the rover "remembers" the last allocation point, reducing fragmentation and search overhead for rapid allocations (common in per-frame actor spawning or texture loading). This is a classic 1990s optimization where linear searches were expensive.

**Doubly-Linked List with Sentinels**: The `blocklist` (head/end sentinel) allows efficient insertion/removal without special-case logic at list boundaries—a standard technique predating intrusive data structures.

**Optional Debug Tagging**: Pre/post tags (`MEMORYPRETAG`, `MEMORYPOSTTAG`) enable detecting heap corruption (buffer overflows into adjacent blocks) at runtime. This overhead is compiled out in release builds, showing awareness of performance-critical deployment constraints.

**Purge-Level Tagging**: The `tag` field in `memblock_t` (storing a "purge level") hints at a tiered memory management scheme: less-critical data can be purged when memory is tight, allowing graceful degradation without crashing.

## Data Flow Through This File

1. **Initialization**: Engine startup allocates a `memzone_t` for the main heap (and per-level zones for `LEVELZONESIZE` pools).
2. **Allocation Request**: `Z_Malloc` → searches block list (starting from rover) → finds free block or coalesces fragments → updates rover and returns pointer.
3. **Deallocation**: `Z_Free` → marks block as free (`user = NULL`) → triggers potential coalescing with adjacent free blocks.
4. **Purging**: `Z_Purge` → walks block list, frees blocks below a purge level, making room for new allocations.
5. **Corruption Check** (if enabled): Pre/post tags are checked on every operation, catching overwrites.

## Learning Notes

**Era-Specific Design**: This is idiomatic 1990s DOS game programming. Modern engines use:
- **ECS or object pools** instead of linked-list zones for cache coherence.
- **Bump allocators or generation arenas** for temporary allocations per frame.
- **Standard library allocators** (or custom per-use-case) rather than monolithic zone heaps.

**Key Insight**: The `LEVELZONESIZE` constant (250KB) shows that ROTT explicitly budget memory per level. This is a consequence of DOS memory limits (often <16MB RAM) and the need to stream levels in/out. Modern engines with GBs of RAM handle this differently.

**Rover Pointer Caching**: This optimization is less relevant on modern systems where CPU caches dominate performance, but it's brilliant for a 1990s platform where cache lines didn't exist and RAM access was slow.

## Potential Issues

- **Rover Validity**: If `rover` pointer outlives block deletion or coalescing, it can become a dangling pointer. Not inferable from this header alone, but a common bug in zone allocators.
- **Fragmentation**: First-fit with rover can leave small, unusable fragments between allocations (the `MINFRAGMENT = 64` constant suggests fragments <64 bytes are merged, but gaps between blocks may still reduce usable space).
- **Purge-Level Assumption**: The `tag` field assumes all subsystems respect the purge-level contract. Misuse (allocating game-critical data at a low purge level) could cause crashes.
