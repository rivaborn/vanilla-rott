# rott/r_scale.asm — Enhanced Analysis

## Architectural Role

This function is the **innermost rendering loop** of the software raycasting engine—the tight inner loop executed once per screen column per frame. It bridges the high-level rendering state machine (which sets globals) and raw video memory, implementing the core texture sampling and scaling algorithm that produces the distinctive vertical slice visuals of raycasting games. Called repeatedly by the column-drawing dispatcher (likely in `rt_draw.c`), it is critical to frame rate.

## Key Cross-References

### Incoming (who depends on this file)
- **Caller**: C/C++ rendering pipeline (inferred from `rott/rt_draw.c` based on naming and globals; not directly visible in cross-ref excerpt)
- Expects caller to:
  - Set the seven `_dc_*` globals before each invocation
  - Pre-load `edi` with screen buffer base address
  - Call repeatedly in a loop over screen columns

### Outgoing (what this file depends on)
- **`_ylookup` lookup table** (extern global): Maps screen Y to video buffer offset (key optimization: avoids per-pixel multiply)
- **`_dc_source` texture buffer** (extern global): Pointer to current texture row data; accessed via fixed-point index
- **No function calls**: Entirely inline assembly; zero external function dependencies

## Design Patterns & Rationale

**Self-Modifying Code (patch1, patch2):**  
Per-column scale increment (`_dc_iscale`) is patched directly into two `add ebp, 12345678h` instructions at initialization. Avoids per-pixel register load or memory indirection; trades code safety for 1990s-era pipeline efficiency. The comment "convice tasm to modify code..." (likely "convince") reveals explicit intent.

**Fixed-Point Arithmetic:**  
Texture V-coordinate lives in `ebp` as a 16.16 fixed-point value. Upper 16 bits (`shr ecx, 16`) serve as texture row index; lower bits discarded (no sub-texel filtering). Eliminates floating-point ops on hardware with slow FP units.

**Pixel-Pair Processing:**  
Loop processes two pixels per iteration, alternating write addresses (`edi`, `edi + SCREENWIDTH`). Reduces loop overhead; exploits typical 2-scanline coherence in texture caches.

## Data Flow Through This File

```
Inputs (globals):
  _dc_yl (top Y) ──┐
  _dc_yh (bot Y)  ├──> Pixel count calculation
  _dc_ycenter     ├──> Texture fraction initialization
  _dc_iscale      ├──> Patched into loop
  _dc_texturemid  ┤
  _dc_source      ├──> Texture buffer pointer
  _ylookup        ├──> Screen address offset
  edi (screen base)

Processing:
  1. Compute screen address:  edi + _ylookup[_dc_yl]
  2. Initialize texture frac: _dc_texturemid - (_dc_ycenter - _dc_yl) * _dc_iscale
  3. Iterate ebp += _dc_iscale, fetch _dc_source[ebp >> 16], write pixel

Output:
  Screen memory at [edi + Y*SCREENWIDTH] and [edi + (Y+1)*SCREENWIDTH]
```

## Learning Notes

**1990s Optimization Mindset:**  
This code exemplifies DOS-era engine practice: self-modifying code, precomputed lookup tables (`_ylookup`), fixed-point math, and hand-tuned loops. Modern engines use SIMD, dynamic compile/JIT, and data-oriented design to amortize initialization cost.

**Fixed-Point Texture Coordinates Still Common:**  
The 16.16 fixed-point pattern (integer index from upper bits, fractional part discarded or used for interpolation) is still idiomatic in texture sampling, though now often wrapped in higher-level abstractions or GPU instructions.

**Raycasting Renderer Signature:**  
Vertical column rendering with per-column state (`_dc_*` globals) is characteristic of raycasting engines (Wolfenstein 3D, early Doom-like engines). Modern renderers use projection matrices and rasterization instead.

## Potential Issues

1. **Self-Modifying Code & Modern CPUs:**  
   Instruction cache coherency is not guaranteed on x86 without explicit flushing (`clflush`). On multi-core systems or speculative execution, the patched `add` instructions might execute before the store completes, yielding stale scale values.

2. **No Bounds Checking:**  
   Texture index (`ebp >> 16`) is never clamped. If `ebp` exceeds texture height, arbitrary memory is read. Screen write address (`edi + Y*SCREENWIDTH`) is not validated; overscan writes can corrupt adjacent buffers.

3. **Assumes Linear Video Memory:**  
   Hard-coded `SCREENWIDTH = 96` and linear addressing work for planar 256-color modes but break on modern VRAM layouts.
