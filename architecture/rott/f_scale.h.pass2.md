# rott/f_scale.h — Enhanced Analysis

## Architectural Role

This header is the public interface for the engine's **column-based vertical scaling subsystem**, a fundamental primitive for rendering pre-scaled sprite columns and textured walls. It serves as the bridge between high-level drawing logic (likely in `rt_film.c` based on cross-refs) and performance-critical low-level rendering. The global `cin_*` state variables and register-optimized functions reflect the classic **voxel/column rendering** architecture inherited from early DOS engines like Wolfenstein 3D, where vertical slices of sprites are drawn column-by-column to the framebuffer.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_film.c** — calls `AddEvents` (per cross-ref), likely invokes `R_DrawFilmColumn` and `DrawFilmPost` in its column drawing loop
- **Cinematic subsystem** (`cin_*.c` files: cin_actr.c, cin_evnt.c, cin_util.c) — the `cin_*` global prefix suggests these scaling parameters power cinematic/film rendering

### Outgoing (what this file depends on)
- **Video framebuffer** — `R_DrawFilmColumn` writes to the destination buffer (`buf` parameter, passed via EDI register)
- **Memory/palette subsystem** — reads from `cin_source` (byte buffer) for pixel data
- **Watcom C runtime** — relies on specific register allocation guarantees from the compiler

## Design Patterns & Rationale

**Global State Parameter Pattern**: Instead of passing 6 scaling parameters to each function, the engine maintains a `cin_*` global state block. This is a deliberate tradeoff favoring **call speed** over function purity—critical for a tight inner loop that may execute thousands of times per frame.

**Register-Optimized Calling Convention** (`#pragma aux`): Both functions specify exact register usage and clobbering. This bypasses the Watcom C calling convention entirely, giving the implementer (in `.c`) explicit control over which registers are preserved and which are trashed. Typical of 1990s high-performance graphics code.

**Separation of Concerns**: Two functions serve distinct roles—`R_DrawFilmColumn` (core scaling), `DrawFilmPost` (post-processing/finalization), suggesting a two-pass rendering pipeline.

## Data Flow Through This File

1. **Setup Phase** (upstream, not visible): Caller initializes `cin_yl`, `cin_yh`, `cin_iscale`, `cin_texturemid`, `cin_ycenter`, `cin_source` 
2. **Column Rendering Loop** (multiple iterations):
   - `R_DrawFilmColumn(buf)` reads global state and source pixel data
   - Scales and clips vertical range to `[cin_yl, cin_yh]`  
   - Writes stretched/scaled column to output buffer
3. **Post-Processing**: `DrawFilmPost(buf, src, height)` applies final processing or copy operation with explicit height parameter

## Learning Notes

**Column-Based Rendering**: This exemplifies the column-by-column rendering technique fundamental to early 3D engines. Each vertical "slice" of a sprite is independent, enabling tight SIMD-like optimizations in a scalar CPU era.

**Performance Archaeology**: The Watcom pragmas reveal priorities—latency matters more than modularity. In a modern engine, these would be inline functions with SIMD/vectorized inner loops; here, they're hand-tuned assembly stubs with global state.

**DOS-era Pragmatism**: Passing state via globals rather than parameters saves instruction count and register pressure. This is a valid optimization trade-off for frame-critical code where the global scope is bounded to a single rendering pass.

## Potential Issues

- **Global State**: The `cin_*` globals introduce implicit coupling; if two subsystems try to draw scaled columns concurrently, state will corrupt. However, this is likely safe in a single-threaded, single-frame-at-a-time engine.
- **Register Assumptions**: The pragmas assume Watcom C's register allocation; porting to a different compiler (e.g., GCC, MSVC) would require rewriting both the pragmas and the calling code.
