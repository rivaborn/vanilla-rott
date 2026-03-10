# rott/_z_zone.h

## File Purpose
Private header defining the core data structures for the ROTT engine's zone-based memory allocator. Implements a custom heap with linked-list block management, supporting memory corruption detection for debugging.

## Core Responsibilities
- Define `memblock_t` structure for tracking allocated/free memory blocks in a doubly-linked list
- Define `memzone_t` structure for representing a memory pool with aggregate tracking
- Provide constants for memory management (fragment size, memory limits, DPMI interrupt)
- Support optional memory corruption detection via pre/post tags

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `memblock_t` | struct | Represents a single memory block in the zone. Contains size, user pointer, purge level (tag), and linked-list pointers. Pre/post tags detect heap corruption when `MEMORYCORRUPTIONTEST==1`. |
| `memzone_t` | struct | Represents a zone (memory pool). Tracks total allocated bytes, maintains a doubly-linked list of blocks via `blocklist`, and caches a rover pointer for allocation optimization. |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a type definition header only.

## Control Flow Notes
This header defines data structures used by the memory management subsystem, likely initialized during engine startup. The zone allocator manages dynamic memory allocation throughout the game's lifetime. The rover pointer in `memzone_t` suggests a first-fit allocation strategy with caching for performance.

## External Dependencies
- `develop.h` — provides feature flags (`MEMORYCORRUPTIONTEST`) that conditionally include debug tags in `memblock_t`

## Notes
- Memory corruption detection controlled by compile-time flag to minimize overhead in release builds
- Supports DPMI (DOS Protected Mode Interface) memory operations via `DPMI_INT` constant
- Fragment minimum (`MINFRAGMENT`) and level zone size (`LEVELZONESIZE`) suggest level-specific memory pools
