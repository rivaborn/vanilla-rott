I'll write the second-pass enhanced analysis for `rott/texture.asm`. While the architecture context was truncated, I can infer significant cross-cutting insights from the assembly structure and the era/nature of this code.

# rott/texture.asm — Enhanced Analysis

## Architectural Role
This file implements a critical inner-loop primitive for the software rasterizer's span-filling stage. It's called per-scanline during textured polygon rendering (walls, sprites, surfaces) and performs the actual framebuffer writes for perspective-textured geometry. The tight optimization and self-modifying code placement indicate this is a **hot-path bottleneck** that directly impacts frame rate—typical for 1990s DOS/early Windows rasterizers where CPU-bound pixel plotting dominated.

## Key Cross-References

### Incoming (who depends on this file)
- **Higher-level rasterizer/drawing code** (rt_draw.c, rt_view.c, or similar): Calls `TextureLine_` once per scanline after computing UV start coordinates and deltas via the affine/perspective pipeline.
- **Globals written by RT_VIEW/RT_BUILD subsystem**: Sets `_texture_u`, `_texture_v`, `_texture_du`, `_texture_dv`, `_texture_count`, `_texture_source`, `_texture_dest`, `_texture_destincr` before each call.

### Outgoing (what this file depends on)
- **Texture bitmap memory** (likely in `_rt_draw.h` / tile/sprite cache): Reads 8-bit paletted pixels via `_texture_source` base + computed offset.
- **Framebuffer memory** (display buffer): Writes 8-bit pixels via `_texture_dest` base + per-pixel offsets from `_texture_destincr`.
- **No external function calls** — pure memory I/O with self-contained arithmetic.

## Design Patterns & Rationale

**Self-Modifying Code**: Four immediate operands (patch1–4) are patched at function entry with runtime values. This avoids **register indirect addressing** (`mov [ebp+eax], ...`) in the critical inner loop—a necessary optimization on 486/Pentium where memory addressing modes were expensive relative to immediates. The setup cost is negligible versus the loop count.

**Fixed-Point Arithmetic**: U/V coordinates and deltas are stored as 32-bit fixed-point (V shifted right 10 bits → 64× texture pitch; U shifted right 16 bits). This avoids floating-point operations entirely—crucial on early 90s CPUs with slow/absent FPUs.

**Register Allocation Discipline**: Registers locked by contract (`esi=U`, `ebx=V`, `ecx=count`, `edx=dest`, `edi=offset pointer`). Scratch use of `eax` and `ebp` is minimal, keeping pressure off the 8-register x86 model.

**Tight Loop Unrolling & Alignment**: The `textureloop` is ALIGN'd at 16 bytes (cache line) and has ~10 compact instructions per iteration, minimizing branch mispredicts and I-cache misses on Pentium-era CPUs.

## Data Flow Through This File

```
Setup:  C-globals (_texture_du/dv/source/dest/count/u/v/destincr)
   ↓
Patch:  Immediates in patch1/2/3/4 (setup phase)
   ↓
Loop (per pixel):
   V (fixed-point) → shift right 10, multiply by 64 (texture pitch) → index
   U (fixed-point) → shift right 16 (integer part) → index
   (U + V×pitch) → offset into texture bitmap → 8-bit pixel
   Destination offset (from _texture_destincr[loop_index]) → framebuffer address
   8-bit pixel written to framebuffer
   U += du, V += dv (self-modified adds in patch1/2)
   ↓
Return: Framebuffer modified in-place
```

The **non-linear destination addressing** (`edi` indexing into `_texture_destincr` array) suggests dynamic scanline layouts—possibly supporting screen rotations, reflections, or non-rectangular view ports.

## Learning Notes

**Era-Specific Techniques**:
- This is a **textbook example of 1990s software rasterization**. Modern engines use GPU rasterizers, but this demonstrates:
  - How affine texture mapping (no per-pixel perspective correction) was implemented on the CPU.
  - The importance of register discipline and immediate-mode code generation on register-starved 32-bit x86.
  - Why paletted (256-color) color was standard—8-bit texture lookups are vastly faster than 24/32-bit.

**What Modern Engines Do Differently**:
- **Self-modifying code is now taboo** (CPU pipeline/cache hazards, security mitigations). Runtime constants are passed via function arguments or thread-local state.
- **Fixed-point is rare**; modern CPUs have fast FPUs and vectorization (SSE/AVX) dominates.
- **Span-filling is GPU-resident**; CPU does vertex assembly and draw calls, GPU handles rasterization.

**Connection to Engine Architecture**:
- This is one component of ROTT's **affine texture-mapping pipeline**: compute start UV + deltas (in rt_view.c/rt_draw.c) → call `TextureLine_` per scanline → composite results into framebuffer.
- The contract of `_texture_destincr` being an array of offsets suggests ROTT may support **side-scrolling or rotated views** (common in FPS/action games of that era).

## Potential Issues

1. **Self-modifying code + modern CPU instruction caches**: Patching immediates may not flush I-cache on all CPUs. Likely mitigated by the fact that the patch happens before the loop runs, and the loop runs many times (amortizing the cost). However, if `TextureLine_` is called frequently with different parameters, instruction cache thrashing could occur.

2. **Fixed-point overflow**: The 10-bit V shift and 16-bit U shift assume V and U coordinates fit in the remaining fractional bits. Out-of-range coordinates (e.g., texture wrapping bugs) could cause silent off-by-one or wrapping errors in the texture lookup.

3. **No bounds checking**: If `_texture_destincr` is accessed out-of-bounds or `_texture_count` is corrupted, the function will read/write arbitrary memory. Typical for era, but a stability risk if callers are not carefully validated.
