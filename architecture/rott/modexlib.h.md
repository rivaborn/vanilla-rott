# rott/modexlib.h

## File Purpose
Public header for the ModeX video library, providing VGA register constants, framebuffer state variables, and functions for graphics/text mode switching, buffer management, page flipping, and direct VGA hardware control. ModeX is a specialized VGA planar mode (320×200, 256 colors) commonly used in DOS games of this era.

## Core Responsibilities
- VGA hardware register port definitions (Sequencer, CRTC, Graphics Controller, Attribute, Palette)
- Screen geometry and memory layout constants
- Global video state variables (framebuffer offsets, page pointers, line width, mode flags)
- Mode switching (graphics ↔ text)
- Framebuffer operations (clear, copy, page flipping)
- Vertical blank synchronization
- Direct VGA plane selection via inline assembly wrappers

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| ylookup | int[MAXSCREENHEIGHT] | extern | Scanline offset lookup table; computed once, used to quickly calculate row addresses |
| linewidth | int | extern | Line width in bytes (depends on planar layout) |
| page1start, page2start, page3start | int | extern | Base offsets for up to three framebuffer pages (double/triple buffering) |
| screensize | int | extern | Total size of one screen buffer in bytes |
| bufferofs | unsigned | extern | Current drawing target framebuffer offset |
| displayofs | unsigned | extern | Currently displayed framebuffer offset |
| graphicsmode | boolean | extern | Flag: true if in graphics mode, false in text mode |

## Key Functions / Methods

### GraphicsMode
- Signature: `void GraphicsMode(void)`
- Purpose: Switch VGA to ModeX graphics mode (320×200, planar, 256-color)
- Inputs: None
- Outputs/Return: None
- Side effects: Reconfigures VGA hardware; sets `graphicsmode = true`
- Calls: (implementation in modexlib.c)
- Notes: Inverse of TextMode(); typically called at startup

### TextMode
- Signature: `void TextMode(void)`
- Purpose: Restore VGA to standard text mode
- Inputs: None
- Outputs/Return: None
- Side effects: Reconfigures VGA hardware; sets `graphicsmode = false`
- Calls: (implementation in modexlib.c)
- Notes: Typically called at shutdown

### VL_SetVGAPlaneMode
- Signature: `void VL_SetVGAPlaneMode(void)`
- Purpose: Initialize VGA hardware registers for planar mode operation
- Inputs: None
- Outputs/Return: None
- Side effects: Writes to CRTC and other VGA control registers
- Calls: (implementation in modexlib.c)
- Notes: Part of graphics mode setup

### VL_ClearBuffer
- Signature: `void VL_ClearBuffer(unsigned buf, byte color)`
- Purpose: Fill a framebuffer with a solid color
- Inputs: `buf` (framebuffer offset), `color` (0–3 in planar 256-color)
- Outputs/Return: None
- Side effects: Modifies video memory at offset `buf`
- Calls: Uses VGAMAPMASK to control plane writes
- Notes: Works with 4-plane VGA layout

### VL_ClearVideo
- Signature: `void VL_ClearVideo(byte color)`
- Purpose: Clear currently displayed screen to a solid color
- Inputs: `color` (fill color)
- Outputs/Return: None
- Side effects: Clears video memory at `displayofs`
- Calls: Likely delegates to VL_ClearBuffer

### VL_DePlaneVGA
- Signature: `void VL_DePlaneVGA(void)`
- Purpose: Convert or decompress planar VGA memory (deplanarization)
- Inputs: None
- Outputs/Return: None
- Side effects: Reorganizes video memory layout
- Calls: (implementation in modexlib.c)
- Notes: May convert planar format to linear or vice versa

### VL_CopyDisplayToHidden
- Signature: `void VL_CopyDisplayToHidden(void)`
- Purpose: Duplicate currently displayed page to hidden/back buffer
- Inputs: None
- Outputs/Return: None
- Side effects: Writes to hidden framebuffer page
- Calls: VL_CopyPlanarPage
- Notes: Used in double-buffering workflow

### VL_CopyBufferToAll
- Signature: `void VL_CopyBufferToAll(unsigned buffer)`
- Purpose: Copy one buffer to all framebuffer pages
- Inputs: `buffer` (source framebuffer offset)
- Outputs/Return: None
- Side effects: Writes to multiple pages
- Calls: VL_CopyPlanarPage (multiple times)
- Notes: Used for initialization or full-screen updates

### VL_CopyPlanarPage
- Signature: `void VL_CopyPlanarPage(byte * src, byte * dest)`
- Purpose: Copy one VGA planar page to another within video memory
- Inputs: `src` (source address), `dest` (destination address)
- Outputs/Return: None
- Side effects: Reads and writes video memory
- Calls: VGAREADMAP, VGAMAPMASK (to iterate planes)
- Notes: Handles 4-plane deplanarization internally

### VL_CopyPlanarPageToMemory
- Signature: `void VL_CopyPlanarPageToMemory(byte * src, byte * dest)`
- Purpose: Copy VGA planar video memory to system RAM, deplanarizing it
- Inputs: `src` (video memory address), `dest` (system RAM address)
- Outputs/Return: None
- Side effects: Reads from video memory; writes to system RAM
- Calls: VGAREADMAP (iterates over 4 planes)
- Notes: Converts planar layout to linear; used for screen capture

### XFlipPage
- Signature: `void XFlipPage(void)`
- Purpose: Swap visible and hidden framebuffer pages (page flip / vsync)
- Inputs: None
- Outputs/Return: None
- Side effects: Updates CRTC START registers to change displayed page; swaps `displayofs` and `bufferofs`
- Calls: WaitVBL internally; writes to CRTC_STARTHIGH, CRTC_STARTLOW
- Notes: Double-buffering mechanism; atomic swap at vertical blank

### WaitVBL
- Signature: `void WaitVBL(void)`
- Purpose: Block until vertical blank period begins (vsync)
- Inputs: None
- Outputs/Return: None
- Side effects: Polling loop; waits on STATUS_REGISTER_1 (0x3DA)
- Calls: (busy-wait on hardware register)
- Notes: Prevents visual tearing; called before page flips

### TurnOffTextCursor
- Signature: `void TurnOffTextCursor(void)`
- Purpose: Disable text-mode cursor in graphics mode
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies CRTC_CURSORSTART register
- Calls: (implementation in modexlib.c)
- Notes: Removes cursor flicker in graphics mode

### VGAMAPMASK
- Signature: `void VGAMAPMASK(int x)` — inline assembly
- Purpose: Select which of 4 VGA bitplanes receive write operations
- Inputs: `x` (plane bitmask: 0x0–0xF; each bit = one plane)
- Outputs/Return: None
- Side effects: Writes to SC_MAPMASK register via I/O port 0x3C5
- Calls: (inline asm: `out dx, al`)
- Notes: Critical for planar mode rendering; controls write destinations

### VGAREADMAP
- Signature: `void VGAREADMAP(int x)` — inline assembly
- Purpose: Select which single VGA bitplane to read from
- Inputs: `x` (plane index 0–3)
- Outputs/Return: None
- Side effects: Writes to GC_READMAP register via I/O port 0x3CF
- Calls: (inline asm: `out dx, ax`)
- Notes: Only one plane can be read at a time; used during buffer copies

### VGAWRITEMAP
- Signature: `void VGAWRITEMAP(int x)` — inline assembly
- Purpose: Alternative mechanism to set write plane mask
- Inputs: `x` (plane selection)
- Outputs/Return: None
- Side effects: Writes to port 0x3C5
- Calls: (inline asm: `out dx, al`)
- Notes: Functionally similar to VGAMAPMASK; Watcom inline syntax

## Control Flow Notes
**Initialization:** GraphicsMode() → VL_SetVGAPlaneMode() → initialize ylookup and page pointers.

**Per-frame rendering:** Draw to `bufferofs` using VGAMAPMASK/VGAREADMAP for plane iteration → WaitVBL() → XFlipPage() to atomically swap `displayofs` and `bufferofs`.

**Shutdown:** TextMode() restores text mode.

The ylookup table is precomputed once during init and reused every frame to avoid repeated multiplication when calculating scanline offsets.

## External Dependencies
- `rt_def.h`: Base types (byte, boolean, int), game constants (MAXSCREENHEIGHT, MAXSCREENWIDTH, etc.)
- VGA hardware: Direct I/O ports in range 0x3C0–0x3DA (Sequencer, CRTC, Graphics Controller, Attribute, Palette, Status)
- Implementations: All function bodies reside in modexlib.c (not in this header)
