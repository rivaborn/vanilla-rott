# rott/rt_eng.asm

## File Purpose
Hand-optimized x86 32-bit assembly implementing ray-casting visibility detection for a tile-based game engine. Determines line-of-sight by traversing a grid, marking visible tiles and detecting opaque obstacles (walls/doors).

## Core Responsibilities
- Implement tight ray-casting loop for performance-critical visibility checks
- Traverse tile grid in two directions (X-major, Y-major stepping)
- Mark tiles as visible in spotvis array during traversal
- Call opaqueness checker (IsOpaque_) when non-empty tile encountered
- Handle two casting modes: standard line-of-sight and door-piercing variant
- Write final ray offset to _rc_off on termination

## Key Types / Data Structures
None (assembly file; uses external arrays passed by reference).

| Name | Kind | Purpose |
|------|------|---------|
| _spotvis | external DWORD array | Visibility bitmap for traversed tiles |
| _mapseen | external DWORD array | Map-wide visibility cache |
| _tilemap | external DWORD array | Tile data (indexed by grid position) |
| _rc_off | external DWORD | Output: final ray offset on hit |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| temphold | DWORD | static | Unused debug temporary (dead code) |

## Key Functions / Methods

### RayCast_
- **Signature:** `void RayCast_(void)` — parameters passed in registers (edi, eax, ebx, ecx, edx, esi)
- **Purpose:** Perform grid traversal along a ray, marking visible tiles until hitting an opaque obstacle or door.
- **Inputs:**
  - `edi`: increment value (sign bit determines Y vs. X primary axis)
  - `eax`: Y-tile step (step when advancing Y)
  - `ebx`: Y-tile step magnitude
  - `ecx`: X step for primary loop
  - `edx`: Y step for primary loop
  - `esi`: packed tile coordinate (xtile << 7 + ytile)
- **Outputs/Return:**
  - `_rc_off`: final grid offset where ray stopped
  - implicit: marks _spotvis and _mapseen entries along path
- **Side effects:**
  - Writes to global arrays _spotvis, _mapseen, _rc_off
  - Calls IsOpaque_ for each non-empty tile
- **Calls:**
  - `IsOpaque_` (external, line-of-sight check)
- **Notes:**
  - Two main loops: castloop (handles both X/Y axes via edi sign), and doorcastloop (alternate path for opaque value 2, likely closed doors)
  - Uses x86 addressing tricks: `_tilemap+esi*2` exploits word indexing; `_spotvis+esi` uses byte indexing
  - Commented-out temphold assignments suggest evolution (possibly incomplete refactor)
  - SETFLAG is a macro that tests a register against itself to set CPU flags conditionally
  - Exits on first opaque obstacle (non-zero IsOpaque_ return) in castloop, or first opaque=2 in doorcastloop

## Control Flow Notes
Entry point: `RayCast_` is called from higher-level visibility system (presumably C function calling this assembly stub). The function does **not return in normal C sense** — it writes result to global _rc_off and uses direct `ret` instruction. Fits into frame/visibility-check phase of game loop, likely called per enemy or projectile LOS query.

Two variants:
1. **castloop / topcast / bottomcast**: Standard ray-cast; stops on any opaque tile (IsOpaque_ != 0)
2. **doorcastloop / doortopcast / doorbottomcast**: Secondary pass through doors; stops only when IsOpaque_ returns exactly 2 (door opaque value)

## External Dependencies
- **IsOpaque_**: External function (defined elsewhere); tests if tile index is vision-blocking
- **_spotvis, _mapseen, _tilemap, _rc_off**: External globals (defined elsewhere); visibility and map state
