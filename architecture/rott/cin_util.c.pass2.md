# rott/cin_util.c — Enhanced Analysis

## Architectural Role

This file provides low-level palette I/O abstraction for the cinematic (FMV) subsystem. It sits at the hardware-abstraction boundary between the cinematic playback engine and VGA display hardware, enabling palette swaps during cutscene animation. The functions are called during active cinematic frame updates to synchronize palette state between decoded FMV frames and the video card's 256-color palette registers.

## Key Cross-References

### Incoming (who depends on this file)
- **Cinematic playback pipeline**: Called from the cinematic subsystem (`cin_main.c`, `cin_glob.c`) during FMV frame rendering
- **Declared in**: `rott/cin_util.h` (public interface to cinematic system)
- These functions are the **only** palette-write path during cinematics (palette isn't modified during normal gameplay based on architecture)

### Outgoing (what this file depends on)
- **VGA hardware abstraction**: `modexlib.h` (defines `PEL_READ_ADR`, `PEL_WRITE_ADR`, `PEL_DATA` port constants)
  - These map to standard VGA palette index/data ports (typically 0x3c8/0x3c9 for writes, 0x3c7/0x3c9 for reads)
- **C runtime**: `conio.h` (provides `inp()`/`outp()` for raw I/O port access—DOS-era specific)
- **Memory debugging**: `memcheck.h` (included but unused; suggests memory tracking infrastructure)

## Design Patterns & Rationale

**Direct Hardware Abstraction Pattern**: Rather than buffering palette operations or deferring them, these functions perform synchronous I/O port writes/reads. This is typical of 1990s DOS game engines where:
- No graphics API layer (DirectX 5 was in early beta; most DOS games accessed hardware directly)
- VGA palette registers are memory-mapped or I/O port-accessible
- Synchronous I/O cost was acceptable for palette-upload time (768 bytes = microseconds on ISA bus)

**Bit-Shift Scaling (6-bit → 8-bit)**: VGA DAC internally uses 6-bit intensity per color channel (0–63); applications store 8-bit values (0–255). The `<<2` (read) and `>>2` (write) operations convert between them:
- **Read**: `6-bit_value << 2` expands 0–63 → 0–252 (loses 2 LSBs precision)
- **Write**: `8-bit_value >> 2` truncates 0–255 → 0–63
- This design trades precision for compatibility with existing 256-color palette buffers

## Data Flow Through This File

```
Incoming:
  FMV frame decode → palette buffer (256 colors × 3 RGB bytes = 768 bytes)
                  ↓
          CinematicSetPalette()
                  ↓
          VGA port I/O (0x3c8, 0x3c9)
                  ↓
          Video card DAC registers
                  ↓
          Display (palette lookup for pixel output)

Outgoing (read path):
  VGA port I/O (0x3c7, 0x3c9)
                  ↓
          CinematicGetPalette()
                  ↓
          Caller's 768-byte buffer
          (used for palette snapshots or fade calculations)
```

## Learning Notes

**Idiomatic to 1990s DOS Graphics**:
- Raw I/O port access is the norm; no graphics API indirection
- Palette operations are **synchronous and blocking** (no async queuing)
- The fixed 768-byte buffer size (256 colors × 3) reflects the VGA 256-color palette as a fundamental limitation

**Modern Contrast**:
- Modern engines use graphics APIs (OpenGL, Vulkan, D3D12) that abstract palette access entirely
- Palette swapping is rarely used; instead, dynamic texture lookups or per-pixel shaders handle color mapping
- If palettes are needed (retro games, procedural graphics), they're uploaded as 2D textures, not via I/O ports

**Engine-Specific Pattern**:
- The `cin_util` naming suggests cinematic-only use, distinct from gameplay palette management
- Implies **cinematic frames have independent color tables** from the main game world
- This separation prevents cinematic art from clashing with in-game HUD or player character colors

## Potential Issues

1. **No error handling**: `inp()`/`outp()` calls don't validate success or timeout. If the VGA card doesn't respond, the function silently fails.
2. **Precision loss**: The 6-bit → 8-bit conversion discards 2 bits of intensity data on write, then re-reads may not recover exact original values.
3. **Hardcoded port numbers**: Constants are defined in `modexlib.h`, making hardware retargeting (non-VGA cards) difficult without recompilation.
4. **No synchronization**: Palette writes aren't synchronized to vertical blanking; mid-frame palette changes could cause visible tearing (though less critical for FMV, which updates rarely).
