# rott/fli_main.c — Enhanced Analysis

## Architectural Role

This file implements the **cinematic playback subsystem** for ROTT's FLI/FLC animation format. It sits between the **cinematic scripting engine** (cin_glob.c, cin_main.c) and the **low-level screen renderer** (Screen), translating format-specific decompression chunks into pixel/palette commands. Unlike the main 3D viewport, cinematics use direct, uncompressed screen writes for real-time video playback—making this module a specialized, high-throughput rendering path optimized for pre-baked animation.

## Key Cross-References

### Incoming (who depends on this file)
- **cin_main.c** / cinematic scripting: Likely calls `PlayFlic()` to trigger cinematic sequences  
- **rt_main.c** / game loop: Title screens, interludes, cutscenes invoke `PlayFlic()`
- **rt_menu.c** / UI: Possibly plays splash animations during menus
- *(Specific callers not visible in provided xref; inferred from API design)*

### Outgoing (what this file depends on)
- **cin_glob.h/c**: `GetCinematicTime()`, `CinematicAbort()` — timing clock and user abort flag
- **cin_util.h**: Likely used for palette sync (though not directly called in code shown)
- **Screen subsystem** (unknown module): `screen_open()`, `screen_close()`, `screen_put_dot()`, `screen_copy_seg()`, `screen_repeat_one()`, `screen_repeat_two()`, `screen_put_colors()`, `screen_put_colors_64()`, `screen_width()`, `screen_height()`
- **Machine struct** (rt_def.h?): Bundles Screen + timing state
- **File I/O**: `file_open_to_read()`, `file_read_big_block()`, `lseek()`, `close()`
- **Memory allocation**: `big_alloc()`, `big_free()` for frame data buffering
- **Error reporting**: `Error()` logging

## Design Patterns & Rationale

1. **Dual-Mode I/O Abstraction** (`SetupFlicAccess`, `CopyNextFlicBlock`, `SetFlicOffset`)  
   - Decouples file vs. memory-buffer I/O behind a common interface  
   - Enables preloading FLI into RAM (e.g., for smooth title sequences) or streaming from disk  
   - Rationale: 1995-era disk I/O was unpredictable; buffering critical for latency-sensitive playback

2. **Chunk-Type Dispatch via Function Pointers**  
   - `decode_frame()` switches on `ChunkHead.type`, invoking specialized decoders  
   - Silently ignores unknown chunks (forward-compatible)  
   - Rationale: FLI format may vary; extensible without recompilation

3. **Callback Pattern for Color Output** (`ColorOut` function pointer, `decode_color()`)  
   - Single decompression logic serves both COLOR_256 (0–255) and COLOR_64 (0–63)  
   - Callback handles range normalization: `screen_put_colors` vs. `screen_put_colors_64`  
   - Rationale: Avoids code duplication; cleaner than conditional branches

4. **Goto-Based State Machine in `decode_delta_flc()`**  
   - Complex line-packing format with skip-lines, EOL markers, and dynamic line counts  
   - Goto enables non-blocking line-by-line processing  
   - Rationale: Performance-critical tight loop; avoids function call overhead in 1990s era

5. **Lazy Frame 2 Caching** (`fill_in_frame2()`)  
   - Loop mode caches offset of frame 2 to avoid re-reading frame 1 header on each loop  
   - Rationale: Minimize seek operations; frame 2 may be far from frame 1 on disk

## Data Flow Through This File

```
PlayFlic() entrypoint
  ├─→ flic_open(file/buffer)
  │    ├─→ SetupFlicAccess() — init file handle or buffer offset
  │    ├─→ CopyNextFlicBlock() — read header
  │    └─→ SetFlicOffset(oframe1) — seek to first frame
  │
  ├─→ [flic_play_once() or flic_play_loop()]
  │    └─→ Loop: flic_next_frame(screen)
  │         ├─→ CopyNextFlicBlock() — read FrameHead + chunk data
  │         ├─→ big_alloc() — temporary frame buffer
  │         ├─→ decode_frame(screen) — dispatch chunks
  │         │    ├─→ decode_color_*() → ColorOut callback → screen_put_colors()
  │         │    ├─→ decode_delta_flc/fli() → screen_copy_seg(), screen_repeat_*()
  │         │    ├─→ decode_byte_run() → screen_copy_seg(), screen_repeat_one()
  │         │    ├─→ decode_literal() → screen_copy_seg()
  │         │    └─→ decode_black() → screen_repeat_two(), screen_put_dot()
  │         ├─→ big_free()
  │         │
  │         Sync timing:
  │         ├─→ calc_end_time() — convert ms to cinematic clock ticks
  │         └─→ wait_til(end_time) — busy-wait + CinematicAbort() poll
  │
  └─→ flic_close() — cleanup
```

**Key insight**: Decompression and timing are tightly coupled; each frame is fully decoded before waiting, ensuring predictable frame timing (no jitter from variable decode times).

## Learning Notes

### Idiomatic Patterns (1990s Game Dev)
- **Inline decompression**: No intermediate abstraction; decoding writes directly to screen buffer (vs. modern engines that decode to intermediate texture)
- **Busy-wait polling**: `wait_til()` spins until timeout or abort flag; no interrupts/callbacks for keyboard  
- **Static temporary buffers**: Frame data allocated/freed per frame; no persistent texture cache
- **Format-specific optimizers**: Separate `decode_delta_flc()` and `decode_delta_fli()` (not unified); exploits known constraints

### Modern Comparisons
- **Modern approach**: Decode to intermediate framebuffer → GPU texture → composite into scene  
- **ROTT approach**: Decode directly to screen buffer (simpler, but inflexible once rendered)
- **Modern timing**: Vblank interrupts or frame-rate-independent delta time  
- **ROTT approach**: Busy-wait on cinematic clock (blocks CPU; cannot render HUD/overlays simultaneously)

### Connections to Game Engine Concepts
- **Video codec abstraction**: FLI/FLC is a self-contained, frame-independent format (vs. modern video codecs like H.264 that require sequence parsing)
- **Streaming I/O**: File/buffer dual-mode prefigures modern asset streaming pipelines
- **Direct screen rendering**: Precursor to modern offscreen rendering and post-processing (though limited to opaque fullscreen playback)

## Potential Issues

1. **Synchronous frame timing blocks CPU**  
   - `wait_til()` busy-spins; prevents other engine systems (AI, networking) from running during playback  
   - Mitigated by: cinematics typically occur in menus or between levels, not mid-gameplay

2. **Temporary frame buffer allocation per frame**  
   - `flic_next_frame()` calls `big_alloc(frame_size)` for each frame; may cause heap fragmentation  
   - Mitigated by: frame sizes bounded by FLI format (typically ≤320×200 pixels); `big_free()` always called

3. **Ignored unknown chunk types**  
   - `decode_frame()` silently skips unrecognized chunks; no warning to user/developer  
   - Risk: Silent corruption if FLC spec extended but decoder not updated

4. **No error recovery in color decompression**  
   - `decode_color()` reads compressed data sequentially; malformed data could read past chunk boundary  
   - Mitigated by: file-level validation in `flic_open()` validates header type/size
