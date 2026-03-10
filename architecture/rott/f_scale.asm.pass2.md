# rott/f_scale.asm — Enhanced Analysis

## Architectural Role

This file implements the inner rendering loop for vertical texture-mapped columns in Rise of the Triad's software 3D renderer. It bridges the high-level raycaster (which determines what texture to draw where) and the low-level framebuffer, performing fixed-point texture filtering with aggressive optimization for the tight inner loop. The functions are called per-visible wall column during each frame's rendering pass.

## Key Cross-References

### Incoming (who depends on this file)
- Called from higher-level rendering code in `rott/` (likely `rt_draw.c` or `rt_film.c` based on naming conventions)
- The globals `_cin_yl`, `_cin_yh`, `_cin_ycenter`, `_cin_iscale`, `_cin_texturemid`, `_cin_source` are written by C-level caller code and read here
- `R_DrawFilmColumn_` is the primary hot-path function; `DrawFilmPost_` handles auxiliary data layout
- Functions are linked as `PUBLIC`, indicating cross-module visibility

### Outgoing (what this file depends on)
- Reads `_ylookup` (DWORD array): a lookup table mapping Y screen coordinates to framebuffer row offsets
  - This is critical for converting (x, y) column coordinates to linear framebuffer addresses
  - Likely built once at initialization (e.g., in `BuildTables` or similar, based on the broader codebase)
- Indirect memory accesses: source texture data via `_cin_source` pointer, framebuffer writes via `edi` register
- No function calls; entirely register-based coordination with caller

## Design Patterns & Rationale

**Self-Modifying Code for Zero-Overhead Parameter Passing**
- The scaling factor `_cin_iscale` is patched directly into the instruction stream at `patch1` and `patch2`
- This avoids register pressure and conditional branching in the inner loop
- Common in 1990s x86 optimization; would be considered dangerous/unmaintainable today

**Fixed-Point Arithmetic for Texture Filtering**
- Texture coordinates are 16-bit fixed-point: high 16 bits = integer index, low 16 bits = fractional component
- Allows subpixel-accurate texture sampling without floating-point overhead
- Scaling (`_cin_iscale`) increments the fractional position; `shr ecx, 16` extracts the integer index

**Loop Unrolling with Instruction Interleaving**
- Processes 4 pixels per iteration (pairs of pairs)
- Instructions are interleaved so memory reads happen well before writes, hiding load latency
- `mov al, [esi+ecx]` (read) is separated from `mov [edi], al` (write) by multiple instructions

**Register Allocation**
- Tight allocation: `esi` = source, `edi` = destination, `ebp` = running texture coordinate, `ecx`/`edx` = scratch
- Reflects the x86 register poverty of the 386/486 era

## Data Flow Through This File

1. **Setup Phase:**
   - Caller writes: `_cin_yl` (column start), `_cin_yh` (column end), `_cin_ycenter`, `_cin_iscale`, `_cin_texturemid`, `_cin_source`, and `edi` (destination framebuffer pointer)
   - `R_DrawFilmColumn_` reads `_ylookup[_cin_yl]` to convert Y coordinate to framebuffer offset
   - Calculates baseline texture coordinate: `ebp = _cin_texturemid - (_cin_ycenter - _cin_yl) * _cin_iscale`

2. **Render Phase:**
   - Loop repeatedly: increment `ebp` by `_cin_iscale`, extract integer index, read pixel from `_cin_source[index]`, write to framebuffer
   - Destination pointer `edi` advances by `SCREENWIDTH*2` (96*2) per iteration to step down two rows
   - Handles odd pixel count edge case at the end

3. **Output:**
   - Modified framebuffer in-place; no return value

**DrawFilmPost_** appears to be a utility for rearranging texture data (interleaving or spreading across vertical strides) — likely used during load time or between frames.

## Learning Notes

- **Era-specific optimization:** Self-modifying code and aggressive hand-tuned assembly were standard in 1990s game engines. Modern systems (caches, branch prediction, JIT) make this approach counterproductive.
- **Fixed-point ubiquity:** All position, scale, and texture-coordinate math uses 16.16 fixed-point. This is idiomatic for that era; floating-point was slow on 486/Pentium.
- **Software rasterization:** This is a raycaster's inner loop, not a triangle-rasterizer. Each column is independent; no triangle setup, no barycentric interpolation.
- **Cache-unfriendly writes:** `DrawFilmPost_` spreads output over `SCREENROW*2` strides, likely for frame interleaving (showing one field per frame in interlaced video mode, which was common in 1990s arcade/console hardware).

## Potential Issues

- **Self-modifying code visibility to CPU caches:** On modern CPUs with separate instruction and data caches, the patching at `patch1`/`patch2` could stall the pipeline or miss the cache line. This worked on 386-era processors but is problematic on modern hardware.
- **Register pressure:** `edi` is assumed to be set by the caller and is not saved; any caller-saved registers must be preserved by the C-level wrapper.
- **Framebuffer layout assumption:** Assumes `_ylookup` table is correctly initialized and that `SCREENWIDTH = 96` matches actual framebuffer stride. Off-by-one errors would cause corruption.
- **No bounds checking:** Trusts that `_cin_source` pointer is valid and that texture indices don't exceed bounds. Buffer overflow risk if `_cin_iscale` or `_cin_yh` are malformed.
