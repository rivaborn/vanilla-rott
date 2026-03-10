# rott/develop.h

## File Purpose
Development and build configuration header for Rise of the Triad. Defines compile-time feature flags to enable/disable debug features, cheats, test modes, and game variants. All flags default to 0 (off) except for production settings.

## Core Responsibilities
- Toggle debug modes (DEBUG, DEVELOPMENT, SOUNDTEST, PRECACHETEST)
- Enable test modes for specific subsystems (ELEVATORTEST, LOADSAVETEST, BATTLECHECK)
- Control cheat codes (WEAPONCHEAT)
- Define game variant (SHAREWARE, SUPERROTT, SITELICENSE)
- Provide debug location tracking macros (wami/waminot)
- Gate UI modes (TEXTMENUS vs. default)

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| programlocation | (inferred int) | global | Tracks current program execution location for debugging; set by `wami()` macro when WHEREAMI=1 |

## Key Functions / Methods
None (preprocessor macros only; see Notes).

## Notes
- **wami(val) / waminot()**: Conditional debug location tracking macros. When `WHEREAMI==1`, `wami(val)` assigns `programlocation=val` to track execution flow; otherwise both macros are empty (no-ops). Intended for runtime debugging breakpoints.
- **Build variant mutual exclusion**: Comments note that only one of (DELUXE, LOWCOST) and only one of (SHAREWARE, SUPERROTT, SITELICENSE) should be enabled simultaneously.
- **Release notes**: BATTLECHECK and BATTLEINFO comments indicate these should be off for release builds, on for beta.
- All flags are 0 except: TEXTMENUS=1, WEAPONCHEAT=1, SYNCCHECK=1, SUPERROTT=1.

## Control Flow Notes
Purely compile-time configuration. Defines are included early to gate feature compilation throughout the codebase. Not involved in frame/render/update cycles.

## External Dependencies
- `programlocation` variable (defined elsewhere)—used by conditional wami macro
