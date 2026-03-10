# audiolib/source/debugio.c — Enhanced Analysis

## Architectural Role

This module provides low-level on-screen debug output for the audio subsystem (and potentially the entire engine) during development and testing. By writing directly to VGA monochrome memory at 0xb0000, it bypasses the DOS/game's normal output layers, making it ideal for debugging initialization sequences, hardware detection, and real-time audio events before higher-level I/O is available. Given its location in `audiolib/` and simple interface, it functions as a shared utility available to all subsystems that need raw diagnostic output in a DOS environment.

## Key Cross-References

### Incoming (who depends on this file)
Based on the cross-reference index provided, the following functions from `debugio.h` are **declared as public**:
- `DB_SetXY`, `DB_PutChar`, `DB_PrintString`, `DB_PrintNum`, `DB_PrintUnsigned`, `DB_printf`

The cross-reference context does not show specific callers, but the public API in `debugio.h` suggests these are available to any subsystem. Typical callers would be:
- Audio initialization code (`BLASTER_Init`, `ADLIBFX_Init`, etc.) for diagnostic output
- Error handling paths (e.g., `ADLIBFX_ErrorString`, `DMA_ErrorString`) for reporting failures

### Outgoing (what this file depends on)
- **C Standard Library**: `<stdio.h>` (EOF constant), `<stdarg.h>` (variadic argument handling), `<stdlib.h>` (included but unused)
- **Hardware**: Direct writes to memory address `0xb0000` (VGA monochrome display adapter)
- **No inter-module dependencies**: No calls to other audiolib or engine subsystems

## Design Patterns & Rationale

**Direct Memory I/O Pattern**: The code bypasses OS/BIOS calls and writes directly to video memory. This is:
- Necessary for DOS real-mode debugging before higher-level I/O is initialized
- Unsafe (assumes hardware layout, crashes in protected mode or on non-VGA systems)
- Practical for 1990s game development where hardware was standardized

**Stateful Cursor Management**: `disp_offset` (static global) tracks cursor position as a byte offset into video memory. This avoids parameter passing and reflects the era's coding style, but couples all functions to shared mutable state.

**Printf-like Variadic Interface**: `DB_printf` mimics standard `printf` but with a minimal format subset (%d, %s, %u, %x). This provides familiar API to developers while avoiding C stdio dependencies.

**Manual Number Conversion**: `myutoa` and `myitoa` reimplement number-to-string conversion instead of using stdlib (e.g., `strtol`). This is:
- Self-contained (fewer dependencies)
- Educational (shows digit-by-digit conversion and radix handling)
- Inefficient (builds string backwards then reverses)

## Data Flow Through This File

```
User code (audio init, error handling, etc.)
  ↓
DB_printf() / DB_PutChar()
  ↓ (parse format string and variadic args)
  ↓
DB_PrintNum() → myitoa() → DB_PrintString()
    or
DB_PrintUnsigned() → myutoa() → DB_PrintString()
    or
DB_PutChar() (directly)
  ↓
Direct writes to 0xb0000 + disp_offset
(updates disp_offset, scrolls screen if needed)
  ↓
Character appears on monochrome display
```

State transitions in `DB_PutChar`:
- Normal char (≥32): write char, advance offset
- `\r`: reset offset to column 0 of current row
- `\n`: advance to next row, clear it
- Offset overflow: scroll screen (copy lines up), reset offset

## Learning Notes

**Bare-Metal I/O**: Demonstrates direct hardware access common in DOS games—no abstraction layers or drivers. The offset calculation `(x * 2) + (y * 160)` reveals VGA memory layout: 160 bytes per line (80 chars × 2 bytes each: char + attribute).

**Integer-to-String Without Stdlib**: The reversal technique in `myutoa` (collect digits in reverse, then copy backwards) is a classic approach predating `snprintf`. Modern engines would use formatted output abstractions or post-process strings.

**Initialization-Free Design**: No explicit `init()` call; `disp_offset` starts at row 24 (off-screen), so first output auto-scrolls to visible area. This reflects the simplicity of early debug utilities.

**Screen Scrolling Logic**: `DB_PutChar` implements line scrolling manually (copy lines 1–23 to lines 0–22), then clears line 23. This is CPU-intensive by modern standards but unavoidable without DMA or hardware support.

**Minimal Format Support**: Unlike full `printf`, only %d, %s, %u, %x are supported. Missing: width specifiers, padding, %c, %f, etc. Reflects the minimalist principle: add only what debugging actually needs.

## Potential Issues

1. **Buffer Overflow Risk** (`myutoa`/`myitoa`): Fixed 100-byte stack buffer. Pathological inputs (e.g., very large numbers) could theoretically overflow, though unlikely with typical debug usage.

2. **Non-Portable Hardware Assumption**: Hardcoded `0xb0000` address and 80×24 display mode. Fails silently on non-VGA systems (286+ protected mode, modern emulators without proper mapping).

3. **No Bounds Checking on Radix**: `DB_PrintUnsigned` accepts any radix but only validates digits 0–35 (A–Z). Radix < 2 or > 36 would produce incorrect output without error.

4. **Thread Unsafe**: Static `disp_offset` shared across all calls; no synchronization. In a multi-threaded or interrupt-driven context (ISR debug output), concurrent calls would corrupt screen state.

5. **Format String Flaw in `DB_printf`**: Truncated format string (e.g., `"...%"`) returns `EOF` but doesn't consume remaining variadic args, risking stack misalignment if `va_end` doesn't clean up (usually safe, but poor practice).

6. **Assumes Text Mode**: No attempt to set video mode or detect current mode; assumes monochrome text mode active. If caller is in graphics mode, writes trash to video RAM.
