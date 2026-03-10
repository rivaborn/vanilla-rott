# rott/fli_glob.h — Enhanced Analysis

## Architectural Role

`PlayFlic` is a gateway into the FLI animation subsystem, positioned as a bridge between high-level game systems (cinematics, intros, menus) and low-level playback machinery (`fli_main.c`). Its dual-mode signature reflects a mid-1990s engineering trade-off: streaming from disk (`usefile=1`) vs. preloading animations into memory buffers for guaranteed playback without stalls. This pattern appears throughout the codebase (see `CopyNextFlicBlock` in `fli_main.h`), suggesting FLI animations are a first-class media type alongside MIDI/PCM audio.

## Key Cross-References

### Incoming (who depends on this file)
- **Cinematic sequencing**: The `cin_*.c` subsystem (CreateEvent, AddEvent, CinematicDelay) likely orchestrates FLI playback as part of scripted sequences
- **Game flow control**: Presumably called from `rt_main.c` during intro/outro cinematics and from `rt_menu.h` for menu animations
- **High-level animation events**: Game state machine (BATTLE, level transitions) probably triggers animations via this interface

### Outgoing (what this file depends on)
- **`fli_main.c/h`**: Houses the actual playback loop and frame decompression (evidenced by `CopyNextFlicBlock`)
- **Graphics/display layer**: Must reach video memory or frame buffer during playback
- **File I/O**: When `usefile=1`, reads animation data from disk (likely integrated with the game's resource system)
- **Palette system**: FLI animations carry embedded palettes; implies coordination with `CinematicSetPalette` / `CinematicGetPalette` (cin_util.h)

## Design Patterns & Rationale

**Dual-mode loading pattern**: The `usefile` / `buffer` split is not stylistic—it's a memory strategy. Pre-buffered animations play without I/O latency (critical for smooth cutscenes), while file-based loading may suffice for lower-priority animations. This was essential on 1990s hardware with slow disk drives.

**Void return type**: The function signature provides no blocking/async hints. Implementation likely either blocks until animation completes or uses interrupt/service-loop mechanisms (idiomatic to the era's fixed timestep game loops).

**Sparse signature**: No error reporting, duration query, or callback mechanism visible—suggesting either silent failure, or error handling is delegated to upper layers or the implementation.

## Data Flow Through This File

```
name (filename/ID) ──┐
buffer (data blob) ──┼──> PlayFlic() ──> fli_main.c decompression
usefile (mode flag)──┤                   ↓
loop (repeat flag) ──┘              Display frames
                                     ↓
                            Palette sync (cin_util.h)
```

The function acts as a **procedural boundary**: caller supplies animation metadata and mode; the implementation handles decompression, timing, and rendering. The `loop` flag suggests the function either internally loops or signals the caller to re-invoke.

## Learning Notes

**Era-specific idiom**: FLI/FLIC (Autodesk Animator) was the *de facto* cutscene format for 1990s PC games—fast decompression on 386/486 CPUs, compact file sizes. Modern engines use H.264/VP9 streaming.

**Memory-speed tradeoff in code**: The dual-mode interface shows a developer mindful of storage vs. latency. Pre-buffering animations (used likely for frequent intros/outros) vs. streaming larger sequences from disk.

**No async abstraction**: Unlike modern engines (which use thread pools for video decoding), this is synchronous—playback blocks the main game loop, implying cinematics were edited to tolerate this constraint.

**Architectural naivety (by modern standards)**: No composition with scene graphs, no frame events for game logic synchronization, no streaming pipeline—just "play animation and return." Suggests cinematics were pre-timed with hardcoded waits elsewhere.

## Potential Issues

- **No return code or error reporting**: Callers cannot detect failed file loads or corrupt data.
- **Blocking guarantee unknown**: If blocking, long animations could cause frame stutters; if async, there's no completion callback visible.
- **Loop semantics unclear**: Does `loop=1` infinitely repeat, or require re-invocation? Unclear from the header.
- **No resource cleanup**: No explicit "stop playback" or palette restoration function declared here (may live in `fli_main.h`).
