Looking at the cross-reference context and first-pass analysis, I can now provide an enriched second-pass analysis. Let me prepare the enhanced document.

# rott/rt_fc_a.asm — Enhanced Analysis

## Architectural Role

This assembly module is the **low-level rasterization backend** for the software 3D renderer. It implements four specialized scanline-drawing routines that are called by the higher-level drawing code (likely `rt_draw.c`'s `CalcRotate`, `CalcHeight`, and related functions). The module handles the innermost pixel-iteration loop for texture-mapped walls, rotated sprites, and sky columns—critical for frame-rate performance in the 1990s-era software renderer.

## Key Cross-References

### Incoming (who depends on this file)

- **rt_draw.c** — calls these functions to render textured walls and sprites
  - `CalcRotate`, `CalcHeight`, and texture-mapping setup functions prepare parameters and invoke these entry points
  - Likely calls `DrawRow_` for front-facing wall sections
  - Likely calls `DrawRotRow_` for rotated/angled walls and sprites
  - Likely calls `DrawMaskedRotRow_` for transparent sprites
  - Likely calls `DrawSkyPost_` for ceiling/sky rendering
- **rt_view.c** — may set up texture coordinates and call `CalcRotate` before invoking assembly
- **lookups.c** — provides lookup tables referenced by texture coordinate math
- **_shadingtable** — external global (color palette LUT, likely set by `rt_draw.c:BuildTables`)
- **_mr_xstep, _mr_ystep, _mr_xfrac, _mr_yfrac, _mr_dest** — external globals set by C caller before each invocation

### Outgoing (what this file depends on)

- **_shadingtable** — color translation palette (256-entry byte LUT, read every pixel)
- **Framebuffer memory** (`edi` destination) — linear byte array at caller-specified address
- **Texture source** (`esi` source) — pre-allocated sprite/wall texture (implicit format: byte-indexed pixels)
- **Global render state** — `_mr_xstep`, `_mr_ystep`, `_mr_xfrac`, `_mr_yfrac` (set once per row/sprite and read during setup)

## Design Patterns & Rationale

**Self-Modifying Code for Parameter Injection:**
- Rather than passing `xstep` and `ystep` in registers (which would require spilling), the code writes them directly into instruction immediates (`hpatch1`, `hpatch2`, etc.)
- **Rationale:** Amortizes parameter setup cost over many pixel iterations; avoids register pressure and indirect addressing overhead
- This was idiomatic in 1990s assembly when instruction-cache efficiency mattered more than debuggability

**Fixed-Point Arithmetic (16.16 Format):**
- Texture coordinates stored as 32-bit fixed-point: high 16 bits = integer part, low 16 bits = fractional part
- SHLD instructions shift fractional part into index bits, masking with 16383 (14-bit texture index)
- **Rationale:** No floating-point hardware on early Pentiums; fixed-point multiplication is 1 cycle

**Pair-Loop Unrolling:**
- Processes 2 pixels per main loop iteration to reduce conditional branch overhead
- Final odd pixel handled separately after loop
- **Rationale:** Halves branch frequency; improves pipeline utilization on Pentium/P6

**Bounds Checking in Rotated Mode:**
- `DrawRotRow_` and `DrawMaskedRotRow_` check x ∈ [0–511], y ∈ [0–255] before texture fetch
- Out-of-bounds pixels default to offset 0 (top-left texel, likely a safe/opaque color)
- **Rationale:** Prevents crashes on rotated sprites near screen edges; simpler than clipping in C

**Masking via Post-Fetch Check:**
- `DrawMaskedRotRow_` fetches pixel first, then checks `cmp bl, 0FFh` before write
- **Rationale:** 0xFF (typically magenta/pink in 256-color palettes) is the transparent sentinel; write-skipping is cheaper than pre-checking and branching before fetch

## Data Flow Through This File

1. **Setup Phase (C caller → Assembly):**
   - C code sets `_mr_xstep`, `_mr_ystep`, `_mr_xfrac`, `_mr_yfrac`, `_shadingtable`, `_mr_dest`
   - C code loads `ecx` (pixel count), `esi` (texture base), `edi` (framebuffer dest) into registers
   - Calls assembly function

2. **Initialization (Assembly):**
   - Unpack fractional coordinates: `ebp` = (yfrac << 16) | xfrac`
   - Self-modify code: write `xstep` and `ystep` into patch sites
   - Calculate first two pixel indices via SHLD + AND (fixed-point to 14-bit index)
   - Fetch first two pixels: `[esi + ecx_index]` and `[esi + edx_index]`
   - Translate through shading table: `al = [eax + al]` (LUT lookup)

3. **Main Loop (Pair iteration):**
   - SHLD: convert fixed-point fractional parts → texture indices
   - ADD `ebp, [xstep_patch]` / `[ystep_patch]`: advance fractional coordinates
   - AND with 16383: mask to valid texture range
   - Fetch pixel from texture
   - Translate color (LUT lookup in `_shadingtable`)
   - Write to framebuffer: `[edi]` and `[edi+1]`
   - Advance `edi` by 2; decrement loop counter
   - Repeat until `loopcount == 0`

4. **Final Pixel (if odd count):**
   - If `pixelcount & 1`, fetch and write one more pixel without advancement

5. **Return:**
   - Framebuffer updated; all state discarded
   - C caller owns reading updated pixels

## Learning Notes

**1. Row-Based Software Rendering (DOOM/Descent Era):**
- This is the scanning heart of a software-only 3D renderer (no hardware accelerator)
- Typical pipeline: C code determines visible row extents → calls assembly for bulk pixel output
- Contrasts with modern GPU shaders where all parameter setup is per-draw-call, not per-row

**2. Fixed-Point vs. Floating-Point:**
- SHLD + fixed-point math (no division, no radix conversion) was essential when FPUs were slow or unavailable
- Modern GPUs use hardware-accelerated floating-point; software renderers today use SIMD (SSE/AVX) integers

**3. Self-Modifying Code:**
- Rare in modern code due to cache coherency complexity and security concerns
- Allowed here because:
  - Single-threaded execution on 1990s systems
  - Patch sites are far from main loop (instruction cache friendly once patched)
  - Eliminates per-row parameter passing overhead

**4. Masking via Sentinel Value:**
- Magic value 0xFF signals transparency; encoding transparency in the palette itself rather than per-pixel alpha
- Limits colors to 255 (lose one palette entry) but requires no extra bits per pixel
- Common in 256-color sprite systems (DOOM, Heretic, Hexen, etc.)

**5. Bounds Checking Strategy:**
- Only `DrawRotRow_` / `DrawMaskedRotRow_` perform checks (rotated sprites need it)
- `DrawRow_` assumes source/dest are pre-validated (straight walls never go out of bounds in BSP renderer)
- Reflects optimization philosophy: shift safety logic to where it's needed

## Potential Issues

- **No Framebuffer Overrun Checks:** Code assumes `edi + pixelcount <= framebuffer_end`. Malformed parameters could corrupt memory.
- **Implicit Format Dependencies:** Assumes texture format (8-bit indexed), shading table format (byte LUT), and framebuffer layout (linear bytes). Mismatch causes visual corruption or crash.
- **No Stack Alignment:** Modern x86-64 ABI requires stack alignment; this code doesn't enforce it (likely not an issue on 32-bit Windows but worth noting for porting).
- **Register Trashing:** Code does not preserve `ebx`, `ecx`, `edx`, `eax`, `esi`, `edi`. Caller must assume all general-purpose registers are clobbered (standard for hot code, but must be documented).
- **Self-Modifying Code Debugging:** Debuggers may not reflect patched instructions; static analysis tools ignore runtime code mutation. Makes profiling and reverse-engineering harder.
- **Cache Misses on Sparse Textures:** If texture data is sparse in memory, cache-line misses per pixel could bottleneck on older Pentiums.
