# rott/rt_dr_a.asm

## File Purpose
Low-level x86-32 assembly rendering routines for the ROTT game engine. Handles VGA video mode setup, screen clearing, and vertical post (wall column) drawing with pixel scaling and shading lookups.

## Core Responsibilities
- Program VGA hardware into 240-pixel-height mode (SetMode240_)
- Clear visibility array and screen buffer (RefreshClear_)
- Draw unmasked wall posts at varying heights and scales (DrawPost_)
- Optimized post rendering with fractional pixel scaling (DrawHeightPost_, DrawMenuPost_, DrawMapPost_)
- Apply shading via lookup tables during scanline rendering
- Handle ceiling/floor fill at screen edges

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| loopcount | DWORD | static | Counter for pixel-pair loops in post drawing |
| pixelcount | DWORD | static | Total pixels in post for final remainder handling |

## Key Functions / Methods

### RefreshClear_
- **Signature:** `RefreshClear_()` → void
- **Purpose:** Clear visibility map and optionally clear screen frame buffer
- **Inputs:** None (reads `_fandc`, `_viewwidth`, `_viewheight`, `_centery`, `_bufferofs`)
- **Outputs/Return:** None
- **Side effects:** Writes 0x1000 dwords to `_spotvis` visibility array; clears screen memory to ceiling/floor colors if `_fandc` is zero
- **Calls:** Direct I/O to SC_DATA, screen memory writes via `rep stosw`
- **Notes:** Early exit if `_fandc` nonzero. Uses `_centery` to split screen into ceiling (top) and floor (bottom) regions. SCREENBWIDE margin calculation for non-aligned width.

### SetMode240_
- **Signature:** `SetMode240_()` → void
- **Purpose:** Program VGA hardware registers for 320×240 video mode
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Direct I/O to VGA sequencer (SC_INDEX/SC_DATA), MISC_OUTPUT, and CRTC (CRTC_INDEX); clears video memory (0xa0000) with `rep stosw`
- **Calls:** Hardware I/O via `out dx, ax/al` and `in al, dx`
- **Notes:** Disables chain4, performs sync reset, configures CRT timing registers (0x0d06, 0x3e07, 0x4109, 0xea10, 0xac11, 0xdf12, 0x0014, 0xe715, 0x0616, 0xe317), clears 32KB of video RAM.

### DrawPost_
- **Signature:** `DrawPost_(ecx: height, esi: src_col, edi: dst_col)` → void
- **Purpose:** Draw a single unmasked wall column with vertical scaling and shading
- **Inputs:** ECX = post height; ESI = source pixels; EDI = destination offset
- **Outputs/Return:** None
- **Side effects:** Writes pixels to screen buffer, modifies code via self-modifying patch at `patch1`
- **Calls:** Reads `_centery`, `_ylookup`, `_shadingtable`; accesses screen memory via EDI/EDX
- **Notes:** Fractional step calculated as `32*65536 / height`; limits drawn height to viewport height; uses row pointers from `_ylookup`; shading lookup via `_shadingtable` (indexed by high byte of pixel values); writes pairs of pixels (high/low bytes of AX).

### DrawHeightPost_
- **Signature:** `DrawHeightPost_(edi: dst, esi: src, ecx: len, ebx: shade_table)` → void
- **Purpose:** Optimized post rendering with fractional source scaling and shading
- **Inputs:** EDI = dest; ESI = source; ECX = post length; EBX = shading table
- **Outputs/Return:** None
- **Side effects:** Writes pixels to screen buffer; self-modifies code at `hp1`, `hp2` labels; reads/writes `loopcount`, `pixelcount`
- **Calls:** Reads `_hp_startfrac`, `_hp_srcstep`, `_shadingtable`
- **Notes:** Renders pairs of pixels per loop iteration; separate calculations for two parallel pixels to hide dependency latency; shading applied via table lookup on pixel index (masked to 63); last odd pixel handled separately. TASM-specific self-modifying pattern.

### DrawMenuPost_
- **Signature:** `DrawMenuPost_(edi: dst, esi: src, ecx: len)` → void
- **Purpose:** Menu/HUD post drawing (similar to DrawHeightPost_ but without shading table lookup)
- **Inputs:** EDI = dest; ESI = source; ECX = post length
- **Outputs/Return:** None
- **Side effects:** Writes raw pixels to screen; self-modifies `mhp1`, `mhp2`; reads/writes `loopcount`, `pixelcount`
- **Calls:** Reads `_hp_startfrac`, `_hp_srcstep`
- **Notes:** Pixel-pair loop identical to DrawHeightPost_ but skips shading table indirection—writes source pixels directly.

### DrawMapPost_
- **Signature:** `DrawMapPost_(edi: dst, esi: src, ecx: len)` → void
- **Purpose:** Overhead map post rendering with unscaled scaling fraction
- **Inputs:** EDI = dest; ESI = source; ECX = post length
- **Outputs/Return:** None
- **Side effects:** Writes pixels to screen; reads/writes `loopcount`, `pixelcount`
- **Calls:** Reads `_hp_srcstep`
- **Notes:** Initializes fraction (EBP) to 0 instead of `_hp_startfrac`; otherwise identical structure to DrawMenuPost_. Used for minimap/map view.

## Control Flow Notes
These functions are framebuffer/raster rendering primitives called during the main game loop's draw phase. RefreshClear_ executes early in frame setup. SetMode240_ runs once during game initialization to configure VGA hardware. DrawPost_ variants are called repeatedly per scanline or per visible wall segment to composite the 3D view. Typical order: SetMode240_ (init) → RefreshClear_ (per frame) → DrawPost_/*Post_ (per visible segment).

## External Dependencies
**Notable includes/macros:**
- `.386p` — 32-bit 386+ instruction set
- `.model flat` — flat memory model (single segment)
- `IDEAL` — TASM pseudo-op for ideal syntax block
- SETFLAG macro — appears to be `test ecx, ecx` or similar

**Defined elsewhere (extern symbols):**
- `_spotvis` — visibility/spotting array (128×128)
- `_viewwidth`, `_viewheight` — viewport dimensions
- `_bufferofs` — framebuffer base offset
- `_fandc` — "fan draw ceiling" flag
- `_ylookup` — row offset lookup table
- `_centery` — viewport center Y coordinate
- `_shadingtable` — palette/shading color translation table
- `_hp_startfrac`, `_hp_srcstep` — fractional scaling parameters
