# rott/modexlib.c

## File Purpose
Mode-X VGA graphics library providing low-level video initialization, buffer management, and page-flipping for DOS 320x200 256-color mode. Handles CPU-to-VRAM transfers, planar memory layout configuration, and synchronization with display hardware.

## Core Responsibilities
- Switch CPU between graphics and text modes via BIOS interrupt 0x10
- Configure VGA hardware registers for non-chained planar mode (Mode-X)
- Manage three video memory pages for double/triple buffering
- Perform synchronization with vertical blank (VBL) timing
- Copy framebuffer data between planar VRAM and linear system memory
- Clear and fill video memory with specified colors
- Execute display page flips with address register updates

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `linewidth` | int | global | Virtual scanline width in words; set by VL_SetLineWidth() |
| `ylookup` | int[MAXSCREENHEIGHT] | global | Lookup table mapping row index to memory offset |
| `page1start` | int | global | Video memory address of first page (0xa0200) |
| `page2start` | int | global | Video memory address of second page (for double buffer) |
| `page3start` | int | global | Video memory address of third page (for triple buffer) |
| `screensize` | int | global | Total bytes per framebuffer (208 × SCREENBWIDE) |
| `displayofs` | unsigned | global | Currently visible framebuffer address |
| `bufferofs` | unsigned | global | Currently writable framebuffer address |
| `graphicsmode` | boolean | global | True if in graphics mode; false if in text mode |

## Key Functions / Methods

### GraphicsMode
- **Signature:** `void GraphicsMode(void)`
- **Purpose:** Switch BIOS video mode to 320×200 256-color mode (0x13).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Issues BIOS int 0x10; sets `graphicsmode=true`.
- **Calls:** `int386()` (interrupt dispatch).
- **Notes:** Raw mode does not enable planar addressing; must call `VL_DePlaneVGA()` afterward for Mode-X.

### TextMode
- **Signature:** `void TextMode(void)`
- **Purpose:** Restore text mode (0x03) via BIOS.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Issues BIOS int 0x10; sets `graphicsmode=false`.
- **Calls:** `int386()`.

### TurnOffTextCursor
- **Signature:** `void TurnOffTextCursor(void)`
- **Purpose:** Hide text mode cursor using BIOS video interrupt.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Sets cursor scan lines to invisible range (0x2000) via int 0x10.
- **Calls:** `int386()`.

### WaitVBL
- **Signature:** `void WaitVBL(void)`
- **Purpose:** Synchronize with vertical blank interval by polling VGA status register.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Busy-waits on port 0x03da; blocks until next VBL.
- **Calls:** `inp()` (port read).
- **Notes:** Polls bits 0x8 of status register; waits for active→inactive→active transition.

### VL_SetLineWidth
- **Signature:** `void VL_SetLineWidth(unsigned width)`
- **Purpose:** Configure virtual screen width in words and populate `ylookup[]` row offset table.
- **Inputs:** `width` — screen width in 16-bit words (typically 48 for 320px @ 4 planes).
- **Outputs/Return:** None.
- **Side effects:** Writes CRTC register 0x13 (OFFSET); populates `ylookup[]`; sets `linewidth`.
- **Calls:** `outpw()` (port write).
- **Notes:** Enables horizontal scrolling via adjusted CRTC offset register.

### VL_SetVGAPlaneMode
- **Signature:** `void VL_SetVGAPlaneMode(void)`
- **Purpose:** Initialize Mode-X plane-based graphics configuration; set up triple buffer.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `GraphicsMode()`, `VL_DePlaneVGA()`, `VL_SetLineWidth(48)`; sets `page{1,2,3}start`, `screensize`, `displayofs`, `bufferofs`; calls `XFlipPage()`.
- **Calls:** `GraphicsMode()`, `VL_DePlaneVGA()`, `VL_SetLineWidth()`, `XFlipPage()`.
- **Notes:** Initializes three 48KB pages at 0xa0200; displayofs = page1, bufferofs = page2.

### VL_CopyPlanarPage
- **Signature:** `void VL_CopyPlanarPage(byte *src, byte *dest)`
- **Purpose:** Copy framebuffer between two locations in video memory, plane-by-plane.
- **Inputs:** `src` — source VRAM address; `dest` — destination VRAM address.
- **Outputs/Return:** None.
- **Side effects:** Loops 4 times (one per plane); sets read/write map registers; calls `memcpy()` for each plane.
- **Calls:** `VGAREADMAP()`, `VGAWRITEMAP()`, `memcpy()` (macros/functions).
- **Notes:** Each plane is 16KB; copies full screen data across planes serially.

### VL_CopyPlanarPageToMemory
- **Signature:** `void VL_CopyPlanarPageToMemory(byte *src, byte *dest)`
- **Purpose:** Convert planar VRAM layout to linear system memory layout.
- **Inputs:** `src` — source in planar VRAM; `dest` — destination in system RAM.
- **Outputs/Return:** None.
- **Side effects:** Loops over 4 planes, 200 rows, 80 bytes; interleaves plane data into destination buffer.
- **Calls:** `VGAREADMAP()`.
- **Notes:** Unplanarizes by reading each plane's byte and writing to offset dest+plane, then stepping by 4 for next pixel.

### VL_CopyBufferToAll
- **Signature:** `void VL_CopyBufferToAll(unsigned buffer)`
- **Purpose:** Copy a single source buffer to all three visible/hidden pages.
- **Inputs:** `buffer` — source buffer address to broadcast.
- **Outputs/Return:** None.
- **Side effects:** Loops 4 planes; conditionally copies to page1, page2, page3 if not equal to source.
- **Calls:** `VGAREADMAP()`, `VGAWRITEMAP()`, `memcpy()`.

### VL_CopyDisplayToHidden
- **Signature:** `void VL_CopyDisplayToHidden(void)`
- **Purpose:** Copy currently displayed page to all non-displayed pages.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `VL_CopyBufferToAll(displayofs)`.
- **Calls:** `VL_CopyBufferToAll()`.

### VL_ClearBuffer
- **Signature:** `void VL_ClearBuffer(unsigned buf, byte color)`
- **Purpose:** Fill a video buffer with a solid color.
- **Inputs:** `buf` — buffer address; `color` — fill byte value.
- **Outputs/Return:** None.
- **Side effects:** Sets VGA map mask to 0x0F (all planes); calls `memset()`.
- **Calls:** `VGAMAPMASK()`, `memset()`.

### VL_ClearVideo
- **Signature:** `void VL_ClearVideo(byte color)`
- **Purpose:** Clear entire 64KB video segment.
- **Inputs:** `color` — fill byte.
- **Outputs/Return:** None.
- **Side effects:** Calls `memset()` on segment 0xa000 (64KB).
- **Calls:** `VGAMAPMASK()`, `memset()`.

### VL_DePlaneVGA
- **Signature:** `void VL_DePlaneVGA(void)`
- **Purpose:** Configure VGA hardware for non-chained (planar) addressing mode.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Disables chain-4 and odd/even modes; switches CRTC to byte addressing; clears entire video memory via `VL_ClearVideo()`.
- **Calls:** `outp()`, `inp()`, `VL_ClearVideo()`.
- **Notes:** Issues register commands to SC_INDEX/SC_DATA, GC_INDEX/GC_DATA, CRTC_INDEX/CRTC_DATA; critical for proper plane setup.

### XFlipPage
- **Signature:** `void XFlipPage(void)`
- **Purpose:** Execute page flip by updating CRTC start address register and advancing `bufferofs` to next page.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Sets `displayofs = bufferofs`; updates CRTC register 0x0C (STARTHIGH) with display address; rotates `bufferofs` through page1→page2→page3→page1.
- **Calls:** `outp()`.
- **Notes:** Triple-buffered; synchronized to avoid mid-frame updates (commented-out `_disable()`/`_enable()` suggest interrupt safety intent).

## Control Flow Notes
This module serves the graphics initialization and frame-rendering pipeline:
1. **Init phase**: `VL_SetVGAPlaneMode()` called once at startup to configure hardware and allocate three framebuffers.
2. **Frame loop**: Game renders to `bufferofs` page; `XFlipPage()` called at end of frame to advance display and buffer pointers.
3. **Utility**: `VL_Clear*()` and `VL_Copy*()` used for buffer management and screen clears.
4. **Sync**: `WaitVBL()` called (externally) to synchronize rendering with display refresh.

## External Dependencies
- **Headers**: `<dos.h>` (BIOS interrupt dispatch via `int386`, `union REGS`); `<string.h>` (`memcpy`, `memset`); `<stdio.h>`, `<stdlib.h>`, `<malloc.h>` (I/O, memory, standard utilities).
- **Local headers**: `modexlib.h` (VGA register constants, macro definitions for `VGAREADMAP`, `VGAWRITEMAP`, `VGAMAPMASK`); `memcheck.h` (memory debugging wrapper—inert in this file).
- **External symbols**: `outp()`, `outpw()`, `inp()` (port I/O); `int386()` (interrupt dispatch); `memcpy()`, `memset()` (memory operations).
