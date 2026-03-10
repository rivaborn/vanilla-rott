# rott/fli_util.h — Enhanced Analysis

## Architectural Role
This file provides the **Hardware Abstraction Layer (HAL)** for the FLI animation playback subsystem. It isolates the FLI decoder (likely in `fli_main.c`) from platform-specific details by abstracting screen rendering, input, timing, and file I/O through a unified `Machine` interface. The original source (Jim Kent's framework, Dr. Dobb's 1993) was widely ported; ROTT adapted it for DOS with multi-segment memory support, making the abstraction critical for cross-platform FLI animation playback during cinematics and splash screens.

## Key Cross-References
### Incoming (who depends on this file)
- `rott/fli_main.c` — FLI decoder/playback engine (referenced as `CopyNextFlicBlock` caller); uses `screen_*`, `clock_*`, `key_*` functions for frame rendering, timing, and input cancellation
- Cinematics system (`rott/cin_*.c`) — likely initializes `Machine` during FLI playback
- **Limitation**: Cross-reference excerpt does not explicitly list all `fli_util.h` function callers; inferred from file structure and naming patterns

### Outgoing (what this file depends on)
- **Base types** (`Uchar`, `Ushort`, `Ulong`, `Boolean`, `ErrCode`, `FileHandle`) — defined in common header (likely `rt_types.h` or similar)
- **Platform-specific implementations** (not visible; presumably in separate `.c` files for DOS/other OS)
- **No visible cross-subsystem calls** within the header itself; functions are stubs/abstractions for platform layer

## Design Patterns & Rationale

**Hardware Abstraction Layer (HAL)**
- Decouples FLI playback from platform details via C function pointers / implementation stubs
- Enables single source tree to support multiple platforms (DOS, Windows, etc.)

**Resource Lifecycle Management**
- Paired `*_open()` / `*_close()` functions (screen, clock, key, machine) — RAII-like pattern in C
- `machine_open()` / `machine_close()` aggregate lifecycle for convenience

**Composite Pattern**
- `Machine` struct bundles `Screen`, `Clock`, `Key` — single entry point for hardware init/shutdown
- Simplifies caller code (e.g., `machine_open()` vs. three separate calls)

**Non-blocking Polling Loop**
- `key_ready()` + `key_read()` split enables game loop to check input without blocking
- Typical of 1990s DOS/console engines before event-driven input

**Segmentation-Aware Memory**
- `MemPtr` and `big_alloc()` / `big_free()` abstract DOS far pointers / segment:offset addressing
- Isolates 64K segment limitation from FLI decoder logic

## Data Flow Through This File
1. **Initialization**: Caller invokes `machine_open()` → opens screen (framebuffer), clock (timer), keyboard in sequence
2. **Frame Loop**: 
   - FLI decoder reads file via `file_read_block()` / `file_read_big_block()`
   - Decodes pixel data into large allocated buffers (`big_alloc()`)
   - Renders via `screen_copy_seg()` or `screen_put_dot()` into framebuffer
   - Updates palette via `screen_put_colors()`
   - Queries timing via `clock_ticks()` for frame pacing
   - Polls input via `key_ready()` / `key_read()` for early exit
3. **Shutdown**: Caller invokes `machine_close()` → restores screen mode, closes clock/keyboard

## Learning Notes
- **Era-specific design**: Explicit hardware abstraction (common in early 1990s engines) — modern engines use middleware (SDL, GLFW) or OS-level abstractions
- **Palette-centric rendering**: 256-color indexed mode (VGA 13h or equivalent) — direct palette writes are a bottleneck by modern standards
- **Segment-aware allocation**: `big_alloc()` handling reflects DOS/real-mode limitations; modern platforms use flat address space
- **Non-blocking input model**: Split `key_ready()` / `key_read()` is idiomatic; contrast modern event-driven input (callbacks, queues)
- **Frame pacing via polling**: `clock_ticks()` suggests manual vsync or frame-rate limiting in caller; no interrupt-driven timing visible
- **File I/O abstraction**: File operations abstracted, suggesting compatibility with different file systems or archived assets

## Potential Issues
- **No error recovery** in lifecycle functions: If `screen_open()` succeeds but `clock_open()` fails, `machine_open()` has no cleanup path (caller must call `machine_close()`; not inferable if this is enforced)
- **No bounds checking** implied in screen functions: `screen_put_dot(x, y)` gives no hint of clipping; caller must validate coordinates
- **Fixed clock speed assumption**: `clock->speed` initialized by `clock_open()`, but no visible way to query or validate; frame timing reliability depends on platform implementation
- **Incomplete interface for large blocks**: `file_read_big_block()` comments say "Could be bigger than 64K" but no verification that reads are atomic across segments; potential data corruption if interrupted
