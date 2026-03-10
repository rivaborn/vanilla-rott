# rottcom/rottipx/global.h — Enhanced Analysis

## Architectural Role
This file is a **foundational platform abstraction layer** for the Rise of the Triad engine, establishing a lowest-common-denominator interface for DOS/x86 hardware I/O. It sits at the base of the dependency tree (widely included by game code, utilities, and networking modules) and enables the engine to operate across multiple subsystems—gameplay (`rott/*`), networking (`rottcom/*`), and build tools (`rtsmaker/*`)—without embedding hardware-specific code throughout the codebase.

## Key Cross-References

### Incoming (who depends on this file)
- **Game runtime**: `rott/rt_util.c`, `rott/rottnet.h` (gameplay and network logic)
- **Networking subsystems**: `rottcom/rottipx/` (IPX layer) and `rottcom/rottser/` (serial/modem layer), each with their own `global.h` variants
- **Build tools**: `rtsmaker/cmdlib.c` (resource compilation)
- **CheckParm** utility is especially widely used across all subsystems for command-line argument parsing

### Outgoing (what this file depends on)
- **Platform libraries** (extern functions): `inp()`, `outp()`, `disable()`, `enable()` — low-level DOS/x86 I/O functions not defined here
- No explicit includes; relies on platform headers to provide x86 interrupt and I/O primitives

## Design Patterns & Rationale

**Platform Abstraction Macro Layer**: Rather than scattering `#ifdef` blocks throughout the codebase, this file centralizes DOS/x86-specific I/O behind portable macros (`INPUT`, `OUTPUT`, `CLI`, `STI`). Callers use abstract names; porting to another platform would only require changing definitions here.

**Minimal Boolean/Type Standardization**: The custom `boolean` enum and sized typedefs (`BYTE`, `WORD`, `LONG`) predate `stdint.h` and `stdbool.h`. This ensures consistent sizes across the engine regardless of compiler variations—critical for network serialization and memory-mapped I/O.

**Modular Global Headers**: The presence of multiple `global.h` files (in `rottipx/`, `rottser/`, and core `rott/`) suggests each subsystem has its own "globals," preventing monolithic header bloat while `rottcom/rottipx/global.h` serves the lowest-level common interface.

## Data Flow Through This File

**Compile-time only**. No runtime state or initialization:
1. Included early in translation units across all subsystems
2. Establishes type aliases and interrupt/port I/O macro bindings
3. Hardware I/O operations (`INPUT`, `OUTPUT`, `CLI`, `STI`) are invoked inline by platform-specific code (DMA, sound card, ISA bus drivers)

## Learning Notes

**What a developer studying this engine learns:**
- **Era-specific patterns**: Direct port I/O (x86 `IN`/`OUT` instructions) and interrupt control were standard for DOS TSR/driver code; this is now encapsulated or absent in modern engines
- **Hardware constant**: `CLOCK_FREQUENCY` (1.8432 MHz) is the x86 PIT (Programmable Interval Timer) base frequency—proof this engine tightly couples to ISA bus hardware
- **Minimal abstraction philosophy**: No wrapper classes or callback registries; interrupts disabled with `CLI()` inline when needed (compare to modern hardware abstraction layers like SDL)

**What's idiomatic to this era:**
- Custom `boolean` type instead of `_Bool` or `bool` 
- Explicit `byte`/`WORD`/`LONG` for cross-platform memory safety (before `uint8_t`)
- Variadic `Error()` mirroring C library conventions (`printf`-style)

## Potential Issues

- **Hardware clock constant never verified as used**: The `CLOCK_FREQUENCY` macro is defined but not present in the cross-reference excerpt; may be dead code or used only in obscure timing code
- **Error() / CheckParm() have no visible callers in cross-ref excerpt**: Either these functions are called extensively (but not indexed in the excerpt) or they are overspecified stubs
- **Interrupt macros (`CLI`, `STI`) are dangerous outside careful critical sections**: No guard against re-entrancy if misused by callers
