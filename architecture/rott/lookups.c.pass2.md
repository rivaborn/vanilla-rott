# rott/lookups.c — Enhanced Analysis

## Architectural Role

This is a **build-time utility** that generates four critical lookup tables for the ROTT raycaster: pixel-to-angle mappings for perspective correction, sine/tangent tables for fast trigonometry, and gamma correction curves for palette-based rendering. Unlike runtime code, `lookups.c` executes once during build, produces a binary artifact (`*.dat`), and that artifact is later loaded by the renderer (likely `rt_draw.c` via `BuildTables`). The file exemplifies the precomputation pattern essential to 1990s real-time 3D graphics: expensive floating-point calculations happen offline, integer lookups happen 60+ times per frame.

## Key Cross-References

### Incoming (who depends on this file)
- **No runtime callers**: This is a standalone utility executable. The functions `BuildGammaTable`, `BuildSinTable`, `BuildTanTable`, `CalcPixelAngles` are invoked only during the build phase via `main()`.
- **Runtime consumers of generated tables**: The `.dat` file produced here is loaded by `rt_draw.c:BuildTables()` (visible in cross-reference index), which deserializes and populates the game engine's runtime lookup arrays.

### Outgoing (what this file depends on)
- **Header constants** from `rt_def.h`, `rt_util.h`, `rt_view.h`:
  - `PANGLES=512`, `FINEANGLES`, `FINEANGLEQUAD` – angular resolution for raycaster
  - `FPFOCALWIDTH` – focal distance for perspective projection
  - `GLOBAL1` (= 1<<16) – fixed-point scale factor
  - `GAMMAENTRIES`, `NUMGAMMALEVELS` – gamma table dimensions
  - `PI` – used for angle calculations
- **C math library**: `sin()`, `tan()`, `atan()`, `pow()` for floating-point computation
- **POSIX/DOS file I/O**: `open()`, `write()`, `close()` for binary serialization
- **Standard library**: `strerror()`, `exit()`, variadic argument handling

## Design Patterns & Rationale

**1. Precomputation + Lookup**
- All four tables are computed from mathematical formulas at build time, then indexed at runtime. This trades build-time cost for runtime speed—critical when a raycaster must perform thousands of angle/trig lookups per frame on 1990s CPUs.

**2. Fixed-Point Arithmetic**
- Sine/tangent values are stored as `fixed` (32-bit) and `short` (16-bit) integers, not floats. This avoids floating-point hardware costs during gameplay. The scale factor `GLOBAL1 = 1<<16` means sine table values are pre-multiplied by 65536, allowing integer multiplication to preserve fractional precision.

**3. Symmetry Exploitation** (in `BuildSinTable`)
- Only 0°–90° is computed; the rest of the circle is derived via mirroring: sin(θ) for quadrant I, then negative/flipped values for quadrants II–IV. This reduces table size by ~75%.

**4. Gamma Curve Generation**
- Generates 8 gamma presets (0x100, 0x120, 0x140, ...). Each is a 64-entry brightness mapping (0–63 input → 0–63 output). Suggests the engine uses 6-bit palette indices and allows dynamic brightness adjustment without reloading graphics.

**5. Chunked Binary Serialization**
- Each table is written as `(int)size + data`. This format allows the loader to validate sizes and detect mismatches. `SafeWrite()` chunks large writes into 32 KB blocks to avoid system call limits on DOS/legacy platforms.

## Data Flow Through This File

```
Command-line arg (filename)
  ↓
main() — validates args, opens output file
  ↓
CalcPixelAngles()  ──→  pangle[512]       (computed from atan, fixed-point)
BuildSinTable()    ──→  sintable[2561]    (computed from sin, mirrored, fixed-point)
BuildTanTable()    ──→  tantable[2048]    (computed from tan, full 360°, fixed-point)
BuildGammaTable()  ──→  gammatable[512]   (computed from pow, 8 curves × 64 entries)
  ↓
SafeWrite() (4 times, each: size prefix + table data)
  ↓
*.dat file (binary)
  ↓
(Later) rt_draw.c:BuildTables() loads *.dat into runtime arrays
```

**Key transformation**: floating-point angles/values → fixed-point integers (multiply by GLOBAL1, truncate) → binary buffer → disk.

## Learning Notes

1. **Precomputation is fundamental to 1990s 3D graphics**: Modern GPUs compute trig on-the-fly; ROTT era engines precomputed everything. This file is the proof.

2. **Fixed-point arithmetic**: Not idiomatic to modern C, but essential for fast integer-only math on CPUs without FPUs. Multiplying two `fixed` values requires an explicit `>>16` shift to maintain precision.

3. **Palette-based rendering**: The gamma table (8 levels × 64 entries) indicates indexed color, not true color. This architecture allowed hardware palette swaps for visual effects.

4. **Pixel angle calculation**: The `+80.0` offset in `CalcPixelAngles()` centers the sample within each pixel (0.5 pixels × 160 pixels/side). This is a classic raycaster technique for perspective-correct projection.

5. **Binary format stability**: The serialized table format (size + data) is brittle—if constants change, the .dat file is invalid. This suggests a hard-coded build dependency; the engine binary probably expects tables in a specific order and size.

## Potential Issues

1. **Hardcoded magic numbers**: Constants like `PANGLES=512`, `FINEANGLES`, gamma starting point `0x100`, increment `32` are defined externally. Changing these requires rebuilding `lookups.c` *and* the engine. No versioning or validation in the binary format to catch mismatches.

2. **Array index assumptions**: `BuildSinTable()` relies on `FINEANGLES`, `FINEANGLEQUAD`, and `FINEANGLES/2` being exact. If `FINEANGLES` is not a power of 2 or these constants don't satisfy `FINEANGLES = 4×FINEANGLEQUAD + 1`, the table will mirror incorrectly. No assertion or validation.

3. **Gamma table off-by-one risk**: The loop `for (l=0; l<NUMGAMMALEVELS; l++, gGamma+=32)` starting at `gGamma=0x100` assumes `NUMGAMMALEVELS` is exactly 8 and increment 32 produces valid gamma values. If `GAMMAENTRIES != 8×64 = 512`, the table will overflow or underfill.

4. **No error handling for math**: If `atan()`, `sin()`, `tan()`, or `pow()` return NaN or Inf, no checks catch it. The fixed-point conversion will silently truncate to garbage.

5. **Utility-only, not part of shipped engine**: This file is build infrastructure, not runtime code. If distributed with the source, developers might accidentally link it into the game binary, inflating size with unused tables.
