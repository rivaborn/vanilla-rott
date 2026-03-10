# rott/rt_ted.h — Enhanced Analysis

## Architectural Role

rt_ted.h serves as the **level initialization gateway** between the resource layer (file I/O, Ted map format) and runtime subsystems (entities, rendering, physics). It orchestrates a linear startup pipeline: load map data → parse metadata → initialize all engine subsystems. This file is central to the multiplayer/team system as well, since `TEAM[MAXPLAYERS]` spawn assignment bridges level design (spawn locations from map) and network/battle modes.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_main.c** (inferred): Calls `LoadTedMap()` + `SetupGameLevel()` during game startup and level transitions
- **rt_game.c** (inferred): Calls game-level initialization; checks exit conditions via `GetNextMap()`
- **rt_battl.h** (from cross-refs): Battle mode integrates team data (`TEAM[]`) for competitive spawns
- **rt_net.h** (from cross-refs): Network subsystem depends on `NUMSPAWNLOCATIONS`, `SPAWNLOC[]` for player placement in multiplayer
- **rt_map.h** (inferred): Map visualization uses `mapwidth`, `mapheight`, `mapplanes[]`
- **rt_stat.h** (from cross-refs): Static entities (walls, lights) initialized by `SetupWalls()`, `SetupLights()`, etc.

### Outgoing (what this file depends on)
- **rottnet.h**: Type definitions (`boolean`, `byte`, `word`, `MAXPLAYERS`)
- **Implied subsystems** (from function names):
  - Asset system: `PreCache*()` functions load sprites/textures (lump system)
  - Entity system: `thingtype` enum for wall entities
  - Door/Switch system: `SetupDoors()`, `SetupSwitches()`, `SetupLinkedActors()`
  - Lighting system: `SetupLights()`, `Illuminate()` (likely per-frame lighting updates)
  - Map format: Relies on `MAPSPOT()` macro and 3-plane structure

## Design Patterns & Rationale

1. **Linear Initialization Pipeline**: Single-threaded startup sequence ensures deterministic entity order and allows dependent setup (e.g., links between clocks). Reflects 1994-95 hardware constraints and network determinism requirements.

2. **Layer-Based Map Storage** (`mapplanes[3]`): Separates tile data, sprites/collision, and metadata—a common approach for 2D tile engines of that era. Avoids complex scene graphs.

3. **Subsystem-Specific Setup Functions**: Each entity type (walls, doors, clocks, lights) gets its own Setup function called from `SetupGameLevel()`. This modular approach isolates subsystem initialization logic but creates tight coupling through globals.

4. **Global State Arrays**: `TEAM[]`, `SPAWNLOC[]`, `walls[]`, `Clocks[]` stored as static globals rather than entity manager. Reflects C-89 style and single-player origins; multiplayer teams added later as parallel array.

5. **Magic Tile Numbers** (e.g., `EXITTILE=107`, platforms via `MAPSPOT(x,y,2)`): Embeds level semantics directly in tile IDs. Hard to extend but simple for Ed-style level editors.

## Data Flow Through This File

```
Disk File (Map format)
    ↓
LoadTedMap(extension, mapnum)
    ↓ (reads into)
mapplanes[3] + mapwidth/mapheight + level metadata (CRC, name)
    ↓
ScanInfoPlane()  [parses info plane for metadata]
    ↓ (populates)
Clocks[], door/switch/light arrays, links
    ↓
SetupGameLevel()  [calls all Setup* functions]
    ├→ SetupWalls()
    ├→ SetupDoors() + SetupDoorLinks()
    ├→ SetupClocks() + SetupLinkedActors()
    ├→ SetupPlayers()  [assigns TEAM[], SPAWNLOC[] to actual player objects]
    ├→ SetupSwitches(), SetupPushWalls(), SetupLights()
    └→ SetupMaskedWalls(), SetupWindows()
    ↓
PreCache()  [load sprites/textures for active level]
    ↓
Illuminate()  [compute/activate lighting]
```

Exit detection happens at runtime: `GetNextMap(tilex, tiley)` checks level transitions when player reaches special tiles.

## Learning Notes

- **Procedural initialization** (not declarative): No config files or data-driven entity definitions; all setup is hard-coded function calls. Contrast with modern engines using JSON/YAML level prefabs.
- **Fixed-size limits** (`MAXWALLTILES`, `MAXCLOCKS=10`, `MAXSPAWNLOCATIONS=50`): Reflects cartridge/early-console memory constraints; would be dynamic arrays in modern code.
- **No hierarchical entity model**: Entities stored in separate subsystem arrays, not a unified scene graph. Setup order is implicit in function call order.
- **Asset precaching by level**: `PreCache()` eagerly loads all sprites/sounds for the level before gameplay starts—appropriate for loading screens but inflexible if level uses many assets.
- **Multiplayer bolted on**: `TEAM[]` and spawn rotation logic feels added post-hoc; single-player spawn was probably simpler (`FIRST`, `SECOND` are hardcoded fallback spawn points?).
- **Map metadata split across structures**: Spawn locations, clocks, links, tile types all derive from scanning the info plane at runtime—no compile-time map validation.

## Potential Issues

1. **Limited extensibility**: Fixed-size arrays and magic tile numbers make it hard to add new entity types or increase map complexity without recompilation.
2. **Implicit subsystem ordering**: Subsystems are initialized in function call order; if one depends on another's state, it's not obvious from the header.
3. **No error recovery**: Map loading failure falls back to `DoPanicMapping()`—a last-resort hack. No validation of map data integrity before setup.
4. **Global state not thread-safe**: Multi-threaded asset loading or concurrent level setup would corrupt global arrays; network code must serialize level changes.
