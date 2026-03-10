# rott/_rt_ted.h

## File Purpose
Private header defining data structures and constants for the TED level editor and map file format. Provides layout constants for precache UI elements and build-time level filename configuration based on game edition.

## Core Responsibilities
- Define map file format structures (`mapfiletype`, `maptype`, `cachetype`)
- Provide macros for actor type checking at map grid positions
- Define precache display layout constants (strings, progress bars, LEDs)
- Conditionally configure level filenames for shareware vs. registered builds
- Define RLEW compression tags and RTL file format version constants

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `cachetype` | struct | Cache entry with lump index and cache level |
| `mapfiletype` | struct | Map file header: RLEW tag, 100 header offsets, tile info array |
| `maptype` | struct | Individual map plane data: 3 plane starts/lengths, width/height, 16-char name |

## Global / File-Static State
None.

## Key Functions / Methods
None. (This is a definitions-only header.)

## Macros & Constants
- **Cache limit:** `MAXPRECACHE` (3500)
- **Actor type checks:** `ActorIsPushWall()`, `ActorIsWall()`, `ActorIsSpring()`, `StaticUndefined()` — test actor grid occupancy
- **Precache UI layout:** `PRECACHEASTRINGX/Y`, `PRECACHEESTRINGX/Y`, `PRECACHESTRINGX/Y`, `PRECACHEBARX/Y`, `PRECACHELED1X/Y`, `PRECACHELED2X/Y`
- **LED count:** `MAXLEDS` (57); **Silly strings:** `MAXSILLYSTRINGS` (32)
- **Format tags:** `SHAREWARE_TAG` (0x4d4b), `REGISTERED_TAG` (0x4344), `RTL_VERSION` (0x0101)
- **File signatures:** `COMMBAT_SIGNATURE` ("RTC"), `NORMAL_SIGNATURE` ("RTL"), `RTL_HEADER_OFFSET` (8)
- **Level filenames** (conditional on `SHAREWARE`, `SUPERROTT`, `SITELICENSE` flags):
  - Registered: "DARKWAR.RTL" / "ROTTCD.RTC" or "ROTTSITE.RTC" or "DARKWAR.RTC"
  - Shareware: "HUNTBGIN.RTL" / "HUNTBGIN.RTC"

## Control Flow Notes
Part of the map/level loading pipeline. Included by map loaders and TED editor integration. The conditional level filename defines are resolved at compile time based on build flags (checked in `develop.h`).

## External Dependencies
- **Includes:** `rt_actor.h` (actor class definitions), `develop.h` (build flags: `SHAREWARE`, `SUPERROTT`, `SITELICENSE`)
- **Symbols defined elsewhere:** `actorat[][]` (actor grid), `sprites[][]` (sprite grid), `objtype` (actor structure), `statobj_t` (static object)
