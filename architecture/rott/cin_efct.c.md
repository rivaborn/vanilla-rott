# rott/cin_efct.c

## File Purpose
Implements cinematic effects rendering for the Rise of the Triad engine. Handles creation, updating, and drawing of interactive cutscene elements: animated sprites, scrolling backgrounds, palette changes, and fade transitions. Provides the core visual pipeline for in-game cinematics.

## Core Responsibilities
- Create and initialize cinematic event objects (sprites, backgrounds, flics, palettes)
- Draw scaled sprites and background layers using column-based fixed-point rendering
- Manage cinematic state updates (animation frames, position interpolation, duration countdown)
- Precache/load cinematic assets from WAD lumps into memory
- Dispatch rendering and update calls based on effect type enum
- Implement fade-to-black and palette transition effects
- Handle VGA plane-based rendering for 320×200 mode

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `flicevent` | struct | Represents a FLI video event (name, loop flag, file vs. lump source) |
| `spriteevent` | struct | Sprite animation with duration, frames, position, scale, and velocities in fixed-point |
| `backevent` | struct | Scrolling background/backdrop with width, offset, and optional composite data buffer |
| `paletteevent` | struct | Palette change event referencing a WAD lump |
| `enum_eventtype` | enum | Enumerates all cinematic effect types (background, sprite, palette, flic, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cin_sprtopoffset` | int | static | Vertical screen offset for scaled sprite column rendering |
| `cin_invscale` | int | static | Inverse scale factor (fixed-point) for scaled post rendering |

## Key Functions / Methods

### SpawnCinematicFlic
- **Signature:** `flicevent * SpawnCinematicFlic(char * name, boolean loop, boolean usefile)`
- **Purpose:** Create and initialize a FLI video event for playback in a cinematic.
- **Inputs:** Lump/file name, loop flag, flag indicating source (file vs. lump)
- **Outputs/Return:** Newly allocated `flicevent` struct
- **Side effects:** Allocates heap memory via `SafeMalloc`
- **Calls:** `SafeMalloc`, `strcpy`
- **Notes:** Stores a shallow copy of the name string (fixed 10-byte buffer).

### SpawnCinematicSprite
- **Signature:** `spriteevent * SpawnCinematicSprite(char * name, int duration, int numframes, int framedelay, int x, int y, int scale, int endx, int endy, int endscale)`
- **Purpose:** Create an animated sprite with keyframe-based movement and scale interpolation.
- **Inputs:** Sprite name, frame count/timing, start/end position and scale (pixels)
- **Outputs/Return:** Newly allocated `spriteevent` struct with computed velocities
- **Side effects:** Allocates heap memory; caches sprite lump to compute dimensions
- **Calls:** `SafeMalloc`, `strcpy`, `W_CacheLumpNum`, `W_GetNumForName`
- **Notes:** Computes fixed-point velocities (`dx`, `dy`, `dscale`) by dividing displacement by duration. Sprite dimensions are fetched but not stored in the event.

### SpawnCinematicBack
- **Signature:** `backevent * SpawnCinematicBack(char * name, int duration, int width, int startx, int endx, int yoffset)`
- **Purpose:** Create a scrolling background with linear pan interpolation.
- **Inputs:** Background name, duration, backdrop width, start/end offsets, vertical position
- **Outputs/Return:** Newly allocated `backevent` struct
- **Side effects:** Allocates heap memory via `SafeMalloc`
- **Calls:** `SafeMalloc`, `strcpy`

### SpawnCinematicMultiBack
- **Signature:** `backevent * SpawnCinematicMultiBack(char * name, char * name2, int duration, int startx, int endx, int yoffset)`
- **Purpose:** Create a composite background by concatenating two image lumps horizontally.
- **Inputs:** Two lump names, duration, scroll parameters, vertical offset
- **Outputs/Return:** Newly allocated `backevent` with merged image data
- **Side effects:** Allocates heap memory; caches and concatenates image data; may call `Error()` if heights mismatch
- **Calls:** `SafeMalloc`, `strcpy`, `W_CacheLumpName`, `memcpy`

### DrawCinematicSprite
- **Signature:** `void DrawCinematicSprite(spriteevent * sprite)`
- **Purpose:** Render a scaled sprite using column-based fixed-point rendering.
- **Inputs:** Sprite event with current position, scale, and frame index
- **Outputs/Return:** None (writes to VGA framebuffer)
- **Side effects:** Sets global `cin_ycenter`, `cin_invscale`, `cin_iscale`, `cin_texturemid`, `cin_sprtopoffset`; calls low-level VGA rendering
- **Calls:** `W_CacheLumpNum`, `W_GetNumForName`, `FixedMul`, `ScaleFilmPost`, `R_DrawFilmColumn`
- **Notes:** Computes screen-space bounding box with clipping; renders only visible columns. Uses fixed-point math to handle sub-pixel positioning.

### ScaleFilmPost
- **Signature:** `void ScaleFilmPost(byte * src, byte * buf)`
- **Purpose:** Helper that scales and renders a sprite column from post data.
- **Inputs:** Pointer to sprite post data, destination VGA buffer pointer
- **Outputs/Return:** None (writes to VGA framebuffer)
- **Side effects:** Sets `cin_yl`, `cin_yh`, `cin_source`; calls `R_DrawFilmColumn`
- **Calls:** `R_DrawFilmColumn`
- **Notes:** Iterates through compressed post format (offset/length pairs terminated by 0xFF); scales vertical positions and lengths by `cin_invscale`.

### DrawCinematicBackground / DrawCinematicMultiBackground
- **Signature:** `void DrawCinematicBackground(backevent * back)`, similar for multi
- **Purpose:** Render a parallax-scrolling background by wrapping columns.
- **Inputs:** Background event with current scroll offset
- **Outputs/Return:** None (writes to VGA framebuffer)
- **Side effects:** Clears buffer if background doesn't fill screen; iterates over VGA planes; calls `DrawFilmPost` for each column
- **Calls:** `W_CacheLumpName`, `DrawClearBuffer`, `DrawFilmPost`
- **Notes:** Implements horizontal wrapping via modulo arithmetic. Single vs. multi versions differ in data source (lump vs. pre-merged buffer).

### DrawCinematicBackdrop
- **Signature:** `void DrawCinematicBackdrop(backevent * back)`
- **Purpose:** Render a patch-based (column-compressed) scrolling backdrop.
- **Inputs:** Background event
- **Outputs/Return:** None (writes to VGA framebuffer)
- **Side effects:** Accesses patch column offset table; calls `DrawFilmPost`
- **Calls:** `W_CacheLumpName`, `DrawFilmPost`
- **Notes:** Uses `collumnofs` (note typo in original) array to index into patch post data.

### UpdateCinematicSprite
- **Signature:** `boolean UpdateCinematicSprite(spriteevent * sprite)`
- **Purpose:** Update sprite state: decrement duration, advance animation frame, interpolate position and scale.
- **Inputs:** Sprite event
- **Outputs/Return:** `false` if duration expired; `true` otherwise
- **Side effects:** Modifies sprite position, scale, and frame fields
- **Calls:** None
- **Notes:** Loops animation frames; uses fixed-point arithmetic for smooth interpolation.

### UpdateCinematicBack
- **Signature:** `boolean UpdateCinematicBack(backevent * back)`
- **Purpose:** Update background: decrement duration, advance scroll offset.
- **Inputs:** Background event
- **Outputs/Return:** `false` if duration expired; `true` otherwise
- **Side effects:** Modifies `duration` and `currentoffset`
- **Calls:** None

### DrawCinematicEffect / UpdateCinematicEffect / PrecacheCinematicEffect
- **Signature:** `boolean DrawCinematicEffect(enum_eventtype type, void * effect)` (and similar)
- **Purpose:** Dispatcher functions that route operations (draw, update, precache) based on effect type enum.
- **Inputs:** Effect type enum and void pointer to effect data
- **Outputs/Return:** Boolean indicating continuation (false for blocking effects like palette/flic/fadeout)
- **Side effects:** Calls type-specific handlers
- **Calls:** Various Spawn*, Draw*, Update*, Precache* functions
- **Notes:** `DrawCinematicEffect` returns `false` for effects that consume a frame (palette, flic, fadeout).

### DrawFadeout
- **Signature:** `void DrawFadeout(void)`
- **Purpose:** Animate a smooth fade to black over 20 VBL frames.
- **Inputs:** None (uses current palette from `CinematicGetPalette`)
- **Outputs/Return:** None (modifies hardware palette)
- **Side effects:** Retrieves and modifies palette; calls `WaitVBL`, `CinematicSetPalette`, `VL_ClearVideo`
- **Calls:** `CinematicGetPalette`, `CinematicSetPalette`, `WaitVBL`, `CinematicDelay`, `VL_ClearVideo`, `GetCinematicTics`
- **Notes:** Interpolates palette values linearly; clears video at end.

### Precache Functions
- `PrecacheFlic`, `PrecacheBack`, `PrecacheCinematicSprite`, `PrecachePalette`
- **Purpose:** Load cinematic assets into memory before playback.
- **Inputs:** Pointer to effect event
- **Outputs/Return:** None
- **Side effects:** Caches lumps via WAD system
- **Calls:** `W_CacheLumpName`, `W_CacheLumpNum`, `W_GetNumForName`

## Control Flow Notes
This module participates in the cinematic playback pipeline:
1. **Precache phase:** Assets are loaded via `PrecacheCinematicEffect` calls.
2. **Per-frame update:** `UpdateCinematicEffect` advances animation state and position; returns `false` when duration expires.
3. **Per-frame render:** `DrawCinematicEffect` renders the effect; returns `false` for frame-blocking effects (palette, flic, fadeout).
4. **Shutdown:** When `cinematicdone` is set by a `cinematicend` event type, playback terminates.

The file uses fixed-point arithmetic (16-bit fraction) throughout for smooth animation interpolation independent of frame rate. Rendering targets a 320×200 VGA mode with 4-plane chunky-pixel layout.

## External Dependencies
- **Cinematic system headers:** `cin_glob.h` (timing), `cin_util.h` (palette), `cin_def.h` (types), `cin_main.h` (extern `cinematicdone`)
- **Graphics/rendering:** `f_scale.h` (scaled column rendering globals and `R_DrawFilmColumn`), `modexlib.h` (VGA mode functions)
- **WAD/lump system:** `w_wad.h` (`W_CacheLump*`, `W_GetNumForName`), `lumpy.h` (patch/lpic structs)
- **Memory:** `z_zone.h` (`SafeMalloc`, `SafeFree`)
- **Format support:** `fli_glob.h` (FLI video)
- **Compiler/debug:** `watcom.h`, `memcheck.h`
- **Standard C:** `string.h` (`strcpy`, `memcpy`), `conio.h` (console I/O)

**Defined elsewhere:**
- `SafeMalloc`, `SafeFree` - memory allocator
- `W_CacheLumpName`, `W_CacheLumpNum`, `W_GetNumForName` - WAD cache
- `CinematicGetPalette`, `CinematicSetPalette` - palette I/O
- `CinematicDelay`, `GetCinematicTics` - timing
- `VL_SetVGAPlaneMode`, `VL_ClearVideo`, `XFlipPage` - VGA control
- `R_DrawFilmColumn`, `DrawFilmPost`, `FixedMul` - low-level rendering
- `Error` - error reporting
- VGA macros: `VGAWRITEMAP`, `VGAMAPMASK`, `bufferofs`, `ylookup`
