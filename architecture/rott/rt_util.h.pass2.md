# rott/rt_util.h — Enhanced Analysis

## Architectural Role

`rt_util.h` is the foundational utility layer of ROTT, providing safe wrappers around system-critical operations that are used throughout the engine. It bridges low-level I/O, memory, and hardware concerns with higher-level game systems: file loading (for assets, maps, saves), memory allocation with level-scoped tracking (crucial for the level-based cleanup pattern), and palette/graphics setup (fundamental to the VGA-based rendering pipeline). The error-handling macros (`SoftError`, `Debug`) act as a compile-time gate controlling whether errors are logged or silently ignored, making this header essential for build configuration across the entire codebase.

## Key Cross-References

### Incoming (who depends on this file)
- **Game initialization & file loading**: Any file loading asset data (maps, sprites, sounds) depends on `LoadFile`, `SaveFile`, and the Safe*Open*/Safe*Read*/Safe*Write* functions. The level system uses `SafeLevelMalloc` for per-level allocations.
- **Command-line parsing**: `CheckParm` is used across multiple subsystems (`rottipx/global.c`, `rottser/global.c`, `rtsmaker/cmdlib.c`), indicating it's the canonical parser for runtime configuration.
- **Networking subsystem**: Likely uses `CheckParm` for protocol configuration; sees a cross-reference in rottipx/global.c.
- **Color/palette rendering**: All graphics drawing code depends on `SetPalette`, `VL_SetPalette`, `BestColor` for palette management and EGA color mapping.
- **Math utilities**: Collision detection and visibility calculations (actor/player subsystems) likely use `FindDistance`, `Find_3D_Distance`, `atan2_appx`.
- **Debuggers/tools**: `MapDebug`, `UL_printf` enable in-game debug output; `CheckDebug` likely gates debug features.

### Outgoing (what this file depends on)
- **C runtime & system calls**: Standard `open/read/write/close`, `malloc/free`, `printf/sprintf`, VGA hardware port I/O (via pragma aux assembly).
- **Build configuration** (`develop.h`): Provides `DEBUG`, `SOFTERROR`, `SHAREWARE`, `SUPERROTT`, `TEXTMENUS` flags that control macro expansion and error handling behavior.
- **Global state**: Reads/writes `egacolor[16]` (EGA palette lookup table), `origpal` (original palette backup), `_argc`/`_argv` (command-line arguments).

## Design Patterns & Rationale

1. **Safe Wrapper Pattern**: Every system call (`open`, `malloc`) is wrapped in a `Safe*` function that invokes `Error()` or `SoftwareError()` on failure. This centralizes error handling and prevents silent failures that would cause corruption or undefined behavior in a game engine running bare-metal graphics modes.

2. **Level-Based Memory Management**: `SafeMalloc` vs `SafeLevelMalloc` suggests a two-tier allocation strategy:
   - `SafeMalloc`: permanent allocations (engine state, lookup tables)
   - `SafeLevelMalloc`: per-level allocations (freed when level transitions)
   - This pattern avoids memory fragmentation and simplifies cleanup during level loads.

3. **Compile-Time Error Gating**: 
   ```c
   #define SoftError  if (1) {} else SoftwareError
   ```
   This macro conditionally disables error reporting at compile time, useful for shipping builds or testing error-recovery paths.

4. **Platform Abstraction via Inline Assembly**: `#pragma aux Square` and `my_outp` encapsulate x86 hardware I/O in a portable way for Watcom C, avoiding scattered inline asm across the codebase. Likely called once during `InitGraphics()`.

5. **Path Manipulation Utilities**: `DefaultPath`, `DefaultExtension`, `ExtractFileBase` follow a modular pattern, composing simple string operations for file system abstraction (portable across Windows/DOS).

## Data Flow Through This File

1. **Initialization Phase**:
   - `CheckParm()` parses `_argc`/`_argv` → game configuration
   - `GetPalette()` / `LoadFile()` → asset loading (palette, maps, sprites)
   - `SafeMalloc()` → engine state allocation
   - `Square()` / graphics mode setup → VGA hardware

2. **Runtime**:
   - Game loop calls math utilities (`FindDistance`, `atan2_appx`) → collision/visibility decisions
   - `SetPalette()` → palette transitions (level change, fade effects)
   - `BestColor()` → dynamic color mapping (e.g., lighting-adjusted colors)
   - Error events → `Error()` or `SoftwareError()` → debug output or soft-error recovery

3. **Level Transitions**:
   - `SafeLevelMalloc()` → per-level allocations
   - `SafeFree()` → cleanup when level ends (invoked by level manager)

4. **Save/Load**:
   - `SaveFile()` → game state → disk
   - `LoadFile()` → game state from disk → `SafeMalloc()`

## Learning Notes

- **Era-specific patterns**: Multi-platform byte-order conversions (`MotoShort`, `IntelShort`) reflect 1990s cross-architecture development (680x0 vs x86); modern engines assume one byte order.
- **Hardware-aware design**: Direct port I/O (`my_outp`, `Square`) and VGA palette manipulation (`VL_SetPalette`) are relics of the DOS/early Windows era; modern engines abstract graphics via APIs (DirectX, OpenGL, Vulkan).
- **Variadic error handling**: `Error(...)` mimics `printf` styling, common in old C codebases before structured logging.
- **Connection to game architecture**: The SafeLevelMalloc pattern hints at ROTT's level-streaming design: each level is a self-contained unit loaded/unloaded independently, enabling content scalability on constrained hardware (32 MB RAM in the '90s).
- **No ECS or scene graphs**: Unlike modern engines, ROTT likely uses imperative actor lists and tile-based maps; utility functions like `FindDistance` and `atan2_appx` are called directly from actor/collision logic rather than via component queries.

## Potential Issues

- **Unbounded error message formatting**: Variadic `Error(...)` and `SoftwareError(...)` calls are vulnerable to format string attacks if user input is passed as the format string. However, this is likely not exploitable in a closed game (no user-facing string input), but worth noting for porting.
- **Endianness conversion functions**: `MotoShort`/`IntelShort` suggest file I/O may read multi-byte data without validation; if file formats are corrupted or tampered, byte-swapping could produce garbage that's used without bounds checking.
- **Level memory tracking**: `SafeLevelMalloc` assumes a proper cleanup mechanism exists; if a level transition is interrupted or a subsystem forgets to free, memory will leak. Hard to verify without seeing the level manager code.
