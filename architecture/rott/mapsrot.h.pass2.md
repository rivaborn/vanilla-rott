# rott/mapsrot.h — Enhanced Analysis

## Architectural Role
This header serves as the **level registry and enumeration schema** for Rise of the Triad's campaign structure. It provides a fixed-size, compile-time enumeration that enables **index-based map lookups and validation** throughout the engine—critical for level loading, game state persistence (save games), menu rendering, and network synchronization in multiplayer modes. The enum values directly correspond to array indices in level data structures (likely stored in mission packs or data files), making this file the bridge between symbolic map names and numerical references.

## Key Cross-References
### Incoming (who depends on this file)
- **Game state/save system**: Code reading/writing player progress or level selections likely uses these enum values to index into map metadata (difficulty, time limits, etc.)
- **Level loader** (`rt_load` family or similar): Validates incoming map IDs against `LASTMAP` before attempting to load resources
- **Menu system** (`rt_menu.h` group): Level selection screens iterate over named maps (excluding EMPTYMAP* entries) for display
- **Networking** (`rt_net.h`): Multiplayer synchronization requires consistent map enumeration across clients; enum values are serialized in game state packets
- **Cheat/debug system** (`rt_debug.c`): CheatMap and related functions use enum values to set/validate current level

### Outgoing (what this file depends on)
- None: This is a pure header with no external dependencies or function calls

## Design Patterns & Rationale
**Fixed-Size Array Indexing with Sentinel Value**: The 84-slot array with EMPTYMAP placeholders reflects a **pre-allocated level table** architecture typical of 1990s console/arcade porting (the game was originally designed for arcade hardware). Rather than dynamic allocation, all possible levels are allocated space at compile time. The `LASTMAP` sentinel (value 83) allows bounds checking: `if (level >= LASTMAP) error()`.

**Preserved Editor Names**: Names like `BURNED_&_AMAZED_MAP`, `HALLSOFFIRE-MV-_MAP` preserve TED5 editor naming directly. This is a "thin wrapper" approach—no abstraction layer for sanitization—suggesting the enum was auto-generated or copy-pasted from TED5 metadata and never intended to be modified in C code.

**Gap Strategy**: Maps 0–35 are filled campaign levels; 37–80 are explicitly reserved empty; 81–82 are bonus levels; 83 is LASTMAP. This layout prevents accidental out-of-bounds access and reserves expansion space, but doesn't grow dynamically.

## Data Flow Through This File
```
[External Code]
    ↓ (reads enum value)
[mapsrot.h: mapnames enum]
    ↓ (used as index)
[Map Metadata Array] (e.g., map properties, difficulty, music)
    ↓ (loads)
[Level Data Resource] (from mission pack file)
    ↓ (populates)
[Game State / Player Progress]
```

The enum itself is **immutable at runtime**—it's compiled into every binary that includes this header. Map selection flows through the enum to validate before resource loading.

## Learning Notes
- **Era-Specific Pattern**: This reflects 1990s game engine design where level counts were fixed at compile time. Modern engines use dynamic level tables or asset pipelines. ROTT's approach mirrors arcade ROM cartridge constraints.
- **TED5 Integration**: The editor names hint at a build pipeline: TED5 level editor → export to C enum → compile → run. No runtime level scripting or hot-loading.
- **Multiplayer Implication**: The fixed enum ensures all networked clients agree on map IDs without runtime negotiation. Each client's binary must have identical map names in identical order.
- **Save Game Compatibility**: If map ordering changes, old save games become invalid (level 5 might load the wrong map). This is a fragile coupling.

## Potential Issues
- **No Version Checking**: Enum ordering is baked into save files and network packets. Adding/reordering maps breaks backward compatibility silently (no validation present).
- **Sparse Array Waste**: 44+ EMPTYMAP slots reserved permanently consume binary space and must be handled in any map-iteration code (conditional checks for `EMPTYMAP*`).
- **Single Sequence Limitation**: No support for multiple campaign branches, modular packs, or dynamic map registration; all levels must be known at compile time.
