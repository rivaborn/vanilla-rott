# rott/_rt_map.h

## File Purpose
Private header for map rendering utilities in the ROTT game engine. Declares an optimized memory-fill function and defines color constants for rendering different map elements (walls, doors, actors, sprites, sky).

## Core Responsibilities
- Define color palette indices for map visualization (`MAP_*COLOR` macros)
- Declare `FastFill()`, an optimized x86 inline-assembly function for rapid memory initialization
- Provide map scaling factor (`FULLMAP_SCALE`)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### FastFill
- **Signature:** `void FastFill(byte * buf, int color, int count);`
- **Purpose:** Optimized byte-fill operation using x86 `rep stosb` instruction; fills a memory buffer with a repeated byte value.
- **Inputs:**
  - `buf` (EDI): pointer to destination buffer
  - `color` (EAX): byte value to fill
  - `count` (ECX): number of bytes to write
- **Outputs/Return:** None (void)
- **Side effects:** Writes to memory starting at `buf` for `count` bytes; modifies ECX register (marked in pragma).
- **Calls:** None (inline assembly only)
- **Notes:** Uses Watcom C++ `#pragma aux` syntax; replaces explicit loops with CPU string instruction for high performance. Likely used during map buffer initialization.

## Control Flow Notes
Map rendering setup phase; `FastFill()` initializes map buffers before drawing geometry. Color constants consumed by rendering functions during frame updates.

## External Dependencies
- **Compiler directive:** `#pragma aux` (Watcom C++ x86 inline assembly)
- No external symbols; self-contained utilities.
