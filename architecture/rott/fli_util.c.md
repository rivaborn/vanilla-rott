# rott/fli_util.c

## File Purpose
Machine-specific abstraction layer for FLI (Flic) animation playback on DOS/VGA hardware. Provides screen display, palette management, keyboard input, clock timing, and file I/O wrappers to isolate platform dependencies from the FLI decoder and renderer.

## Core Responsibilities
- VGA mode switching (mode 0x13: 320×200×8bpp) and graphics initialization via BIOS interrupts
- Direct pixel writing to video memory with horizontal line clipping and bounds checking
- VGA palette (colormap) setup via hardware port I/O (0x3C8–0x3C9)
- Performance-critical rendering using optimized memcpy/memset for common patterns
- Hardware initialization and shutdown orchestration via Machine abstraction
- Wrappers for large-block memory allocation and file I/O (delegate to SafeMalloc/SafeRead)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Pixel | typedef | 8-bit color index (unsigned char) |
| Color | struct | RGB palette entry (r, g, b: 0–255) |
| Pixels2 | struct | Two-pixel word for word-aligned delta encoding |
| Screen | struct | Video mode state, resolution, video memory pointer (0xA0000) |
| Machine | struct | Container for Screen, Clock, Key; top-level hardware abstraction |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| screenlookup[200] | Ushort[200] | static | Pre-calculated y-coordinate to video memory offset (y × 320 bytes) for O(1) address computation |

## Key Functions / Methods

### screen_open
- **Signature:** `ErrCode screen_open(Screen *s)`
- **Purpose:** Initialize VGA mode 0x13 and populate Screen structure with dimensions and video memory pointer.
- **Inputs:** Pointer to Screen struct (cleared on entry).
- **Outputs/Return:** `Success` if mode switch succeeded and verified; `ErrDisplay` if mode 0x13 unavailable.
- **Side effects:** Sets video mode via BIOS (int 0x10); saves old mode; populates `screenlookup[]` table; sets `s->pixels = 0xA0000`.
- **Calls:** `get_vmode()`, `set_vmode()`, `ClearStruct()`.
- **Notes:** Verifies mode was actually set before returning success. If failure, restores original mode before returning error.

### screen_close
- **Signature:** `void screen_close(Screen *s)`
- **Purpose:** Clean up screen state (mode restoration currently commented out).
- **Inputs:** Pointer to Screen struct.
- **Outputs/Return:** None.
- **Side effects:** Clears struct; commented-out call to `set_vmode()` suggests original design would restore video mode.
- **Calls:** `ClearStruct()`.
- **Notes:** Safety mechanism to discourage use-after-close.

### screen_copy_seg
- **Signature:** `void screen_copy_seg(Screen *s, int x, int y, Pixel *pixels, int count)`
- **Purpose:** Copy pixel buffer from RAM to screen at (x, y), with clipping.
- **Inputs:** Screen pointer, x, y, source pixel buffer, pixel count.
- **Outputs/Return:** None.
- **Side effects:** Writes directly to video memory; clips to screen bounds.
- **Calls:** `line_clip()`, `memcpy()`.
- **Notes:** Performance-critical path. Adjusts source pointer offset when clipped. Direct memcpy avoids per-pixel loop overhead.

### screen_repeat_one
- **Signature:** `void screen_repeat_one(Screen *s, int x, int y, Pixel color, int count)`
- **Purpose:** Draw horizontal line of solid color using memset.
- **Inputs:** Screen pointer, x, y, color value, pixel count.
- **Outputs/Return:** None.
- **Side effects:** Writes solid color to video memory; clips to bounds.
- **Calls:** `line_clip()`, `memset()`.
- **Notes:** Optimized for byte-run compression chunk type in FLI format.

### screen_repeat_two
- **Signature:** `void screen_repeat_two(Screen *s, int x, int y, Pixels2 pixels2, int count)`
- **Purpose:** Repeat 2-pixel pattern (word) across screen, optimized for word-aligned writes.
- **Inputs:** Screen pointer, x, y, 2-pixel pattern, word count.
- **Outputs/Return:** None.
- **Side effects:** Writes to video memory in word-aligned chunks; handles odd pixel boundary.
- **Calls:** `line_clip()`.
- **Notes:** Converts count to pixel units for clipping, then back to words. Special case for odd pixel at end (cast to Pixel* and write single byte). Used for DELTA_FLC chunk type.

### screen_put_colors / screen_put_colors_64
- **Signature:** `void screen_put_colors(Screen *s, int start, Color *colors, int count)`; likewise for `_64` variant.
- **Purpose:** Update VGA hardware palette via port I/O.
- **Inputs:** Screen pointer, palette start index, color array, count.
- **Outputs/Return:** None.
- **Side effects:** I/O to ports 0x3C8 (index) and 0x3C9 (RGB data). `screen_put_colors` shifts RGB right 2 bits (0–255 → 0–63); `_64` variant uses values directly.
- **Calls:** `outportb()` (port I/O macro).
- **Notes:** Direct VGA palette write. Used after COLOR_256 or COLOR_64 chunks in FLI frame.

### line_clip (static)
- **Signature:** `static Boolean line_clip(Screen *s, int *px, int *py, int *pwidth)`
- **Purpose:** Clip horizontal line segment to screen bounds; adjust start x and width in-place.
- **Inputs:** Screen pointer, pointers to x, y, width.
- **Outputs/Return:** `TRUE` if any part remains visible; `FALSE` if clipped entirely off-screen.
- **Side effects:** Modifies *px and *pwidth if clipping occurs.
- **Calls:** None.
- **Notes:** Checks y bounds first (fail fast). Assumes horizontal lines only (y is unchanged). Width can become negative after clipping (checked and rejected).

### set_vmode / get_vmode (static)
- **Signature:** `static Boolean set_vmode(Uchar mode)` / `static Uchar get_vmode()`
- **Purpose:** BIOS-level video mode control.
- **Inputs:** set_vmode: desired mode (0x13 for 320×200 256-color).
- **Outputs/Return:** set_vmode always returns TRUE (carry flag check commented out); get_vmode returns current mode.
- **Side effects:** INT 0x10 BIOS interrupt.
- **Calls:** `int86()` (BIOS interrupt macro).
- **Notes:** Uses union REGS for register setup. Original code checked carry flag but it's disabled.

### big_alloc / big_free
- **Signature:** `ErrCode big_alloc(MemPtr *bb, Ulong size)` / `void big_free(MemPtr *bb)`
- **Purpose:** Wrapper for large memory allocation/deallocation (> 64K on real-mode DOS).
- **Inputs:** Pointer to MemPtr, size in bytes.
- **Outputs/Return:** big_alloc returns Success.
- **Side effects:** Calls SafeMalloc/SafeFree (defined elsewhere).
- **Calls:** `SafeMalloc()`, `SafeFree()`.
- **Notes:** Abstraction layer over platform memory management; actual allocation logic defined elsewhere.

### file_open_to_read / file_read_big_block
- **Signature:** `ErrCode file_open_to_read(FileHandle *phandle, char *name)` / `ErrCode file_read_big_block(FileHandle handle, MemPtr bb, Ulong size)`
- **Purpose:** File I/O wrappers for FLI file loading.
- **Inputs:** File name / file handle, buffer, size.
- **Outputs/Return:** Both return Success.
- **Side effects:** File I/O; calls SafeOpenRead/SafeRead.
- **Calls:** `SafeOpenRead()`, `SafeRead()`.
- **Notes:** Abstraction; actual error handling in SafeXxx functions.

### machine_open / machine_close
- **Signature:** `ErrCode machine_open(Machine *machine)` / `void machine_close(Machine *machine)`
- **Purpose:** Initialize / shutdown all hardware subsystems (screen, clock, keyboard).
- **Inputs:** Pointer to Machine struct.
- **Outputs/Return:** machine_open returns Success or error code; machine_close returns void.
- **Side effects:** Calls screen_open (clock and keyboard calls are commented out).
- **Calls:** `ClearStruct()`, `screen_open()`, `screen_close()`.
- **Notes:** Clock and keyboard initialization are conditionally compiled out (`#if 0`). Error chain: keyboard → clock → screen; if screen fails, unwinds (though unwinding is also commented out).

## Control Flow Notes
- **Initialization:** User calls `machine_open()` → sets up VGA mode 0x13 via `screen_open()` → pre-calculates `screenlookup[]` table → ready for frame rendering.
- **Frame rendering:** FLI decoder calls `screen_copy_seg()`, `screen_repeat_one()`, `screen_repeat_two()` for each chunk in the current frame; updates palette via `screen_put_colors()`.
- **Shutdown:** `machine_close()` → `screen_close()` (mode restoration disabled). Clock and keyboard cleanup elided.

## External Dependencies
- **BIOS/Hardware:** `int86()` (INT 0x10 for video, INT 0x1A for timer); `inportb()`, `outportb()` for VGA palette port I/O (0x3C8–0x3C9).
- **Standard C:** `memcpy()`, `memset()`, `<stdlib.h>`, `<mem.h>`, `<dos.h>`, `<bios.h>`.
- **Defined elsewhere:** `SafeMalloc()`, `SafeFree()`, `SafeOpenRead()`, `SafeRead()` (memory and file I/O); `ClearStruct()` macro (fli_main.h); type definitions in fli_type.h.
