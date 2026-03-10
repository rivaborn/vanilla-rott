# rott/rt_film.c

## File Purpose
Cinematic/film playback engine for Rise of the Triad. Parses script files describing timed cutscene events, manages active sprites and backgrounds, and renders animated sequences with scaling, scrolling, palette changes, and fade effects.

## Core Responsibilities
- Parse cinematic script files (`.ms` format) into event queues
- Manage event activation and timing during playback
- Allocate and free actor instances (active event instances)
- Render backgrounds, backdrops, sprites with scaling and scrolling
- Update sprite positions, scales, and lifetimes each frame
- Drive main cinematic playback loop with input handling

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| event | struct | Parsed cinematic event: timing, type, asset name, position/scale, deltas |
| actortype | struct | Active instance of an event: tics remaining, event reference, current position/scale |
| eventtype | enum | Event type: background, backdrop, sprite, palette, fadeout, with optional scrolling variants |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| filmbuffer | byte* | static | Pointer to video buffer for rendering |
| events | event*[MAXEVENTS] | static | Array of parsed events |
| actors | actortype*[MAXFILMACTORS] | static | Array of active event instances |
| eventindex | int | static | Next free slot in events array during parsing |
| currentevent | int | static | Index of next event to activate |
| filmtics | int | static | Current playback time in tics |
| movielength | int | static | Total duration of current film in tics |
| dtime | int | static | Delta time elapsed last frame |
| lastfilmactor | int | static | Highest actor index with allocated instance |
| firsttime | byte | static | Guard flag for one-time initialization |
| dc_ycenter | int | global | Y-center for sprite scaling (used by R_DrawFilmColumn) |

## Key Functions / Methods

### PlayMovie
- **Signature:** `void PlayMovie(char *name)`
- **Purpose:** Main entry point; loads and plays a complete cinematic sequence
- **Inputs:** name (script/lump basename, e.g., "INTRO")
- **Outputs:** None
- **Side effects:** Initializes film state, loads script, runs frame loop until duration/cancel, cleans up
- **Calls:** InitializeMovie, GrabMovieScript, IN_ClearKeysDown, AddEvents, DrawEvents, UpdateEvents, FlipPage, CalcTics, CleanupMovie
- **Notes:** Breaks on ESC key or mouse button; filmbuffer reassigned each frame from bufferofs

### ParseMovieScript
- **Signature:** `void ParseMovieScript(void)`
- **Purpose:** Parse script file into event array; handles timing and event parameter parsing
- **Inputs:** None (reads from global script_p, scriptend_p, token via GetToken)
- **Outputs:** Populates events array, sets movielength
- **Side effects:** Allocates event structures, increments eventindex, accumulates time across tokens
- **Calls:** GetToken, ParseNum, GetNewFilmEvent, strcpy, Error
- **Notes:** Cumulative time: each token is a delta; MOVIEEND marker terminates. Complex if/else chain for event types (BACKGROUND, BACKDROP, SPRITE, BKGNDSPRITE, PALETTE, FADEOUT)

### AddEvents
- **Signature:** `void AddEvents(void)`
- **Purpose:** Activate events whose scheduled time has been reached
- **Inputs:** None (reads filmtics, currentevent)
- **Outputs:** None
- **Side effects:** Allocates and initializes actors from events, increments currentevent, deletes conflicting actors
- **Calls:** GetFreeActor, DeleteEvent, Error
- **Notes:** One-time events (palette, fadeout) get tics=-1; sprites get initial position/scale from event data; DeleteEvent removes previous actors of same type (exclusive behavior)

### UpdateEvents
- **Signature:** `void UpdateEvents(void)`
- **Purpose:** Update all active actors: decrement timers, advance positions/scales, free expired actors
- **Inputs:** None (reads dtime)
- **Outputs:** Updates actor state in-place; frees actors when tics<=0
- **Side effects:** Modifies actor curx, cury, curscale; frees memory via FreeActor
- **Calls:** FreeActor, Error
- **Notes:** Type-specific updates: scrolling backgrounds/backdrops update only curx; sprites update curx, cury, curscale; uses goto continueloop to skip freed actors

### DrawEvents
- **Signature:** `void DrawEvents(void)`
- **Purpose:** Render all active cinematic elements in correct order
- **Inputs:** None
- **Outputs:** Writes to filmbuffer
- **Side effects:** Renders palette/fade → background → background sprites → backdrop → foreground sprites
- **Calls:** DrawPalette, DrawFadeout, DrawBackground, DrawScrollingBackground, DrawSprite, FreeActor
- **Notes:** Draws in layers; palette/fadeout are exclusive (breaks after processing). Uses "done" flag to render only first background/backdrop

### DrawSprite
- **Signature:** `void DrawSprite(char *name, int x, int y, int height, int origheight)`
- **Purpose:** Render a scaled sprite at given position
- **Inputs:** name (lump name), x, y (screen coords), height (scaled height), origheight (original sprite height)
- **Outputs:** None
- **Side effects:** Writes to filmbuffer, sets dc_* globals for column renderer
- **Calls:** W_CacheLumpName, VGAWRITEMAP, ScaleFilmPost, FixedMul
- **Notes:** Fixed-point scaling math (dc_invscale, dc_iscale). Clips to screen bounds [0,320) x [0,200). Calculates column start frac for partial-screen sprites

### ScaleFilmPost
- **Signature:** `void ScaleFilmPost(byte *src, byte *buf)`
- **Purpose:** Scale and render a single sprite column to buffer
- **Inputs:** src (patch column data: offset/length runs), buf (destination buffer)
- **Outputs:** None
- **Side effects:** Writes to buf, sets dc_yl, dc_yh, dc_source for R_DrawFilmColumn
- **Calls:** R_DrawFilmColumn
- **Notes:** Handles scaled vertical spans, clips to screen bounds, skips zero-height spans

### DrawBackground
- **Signature:** `void DrawBackground(char *name)`
- **Purpose:** Render full-screen background from lpic_t lump
- **Inputs:** name (lump name)
- **Outputs:** None
- **Side effects:** Writes to filmbuffer
- **Calls:** W_CacheLumpName, VGAWRITEMAP, DrawFilmPost
- **Notes:** 4-plane VGA iteration; src pointer advanced by (height<<2) per plane

### DrawScrollingBackground
- **Signature:** `int DrawScrollingBackground(char *name, int x)`
- **Purpose:** Render horizontally scrolling background with wraparound
- **Inputs:** name (lump name), x (scroll position in subpixels, range [0, width<<8))
- **Outputs:** Returns updated x position (for storage in actor.curx)
- **Side effects:** Writes to filmbuffer
- **Calls:** W_CacheLumpName, VGAWRITEMAP, DrawFilmPost
- **Notes:** Wraps x at (width<<8); renders columns out-of-order based on xx=x>>8 offset

### DrawBackdrop
- **Signature:** `void DrawBackdrop(char *name)`
- **Purpose:** Render full-screen backdrop from patch_t lump
- **Inputs:** name (lump name)
- **Outputs:** None
- **Side effects:** Writes to filmbuffer via ylookup offsets
- **Calls:** W_CacheLumpName, VGAWRITEMAP, DrawFilmPost
- **Notes:** Patch format: column-major with offset/length runs; topoffset computed once; ylookup used for vertical clipping

### DrawScrollingBackdrop
- **Signature:** `int DrawScrollingBackdrop(char *name, int x)`
- **Purpose:** Render horizontally scrolling backdrop with wraparound
- **Inputs:** name (lump name), x (scroll position)
- **Outputs:** Returns updated x position
- **Side effects:** Writes to filmbuffer
- **Calls:** W_CacheLumpName, VGAWRITEMAP, DrawFilmPost
- **Notes:** Similar to DrawScrollingBackground but uses patch_t format; wraps column indices at p->width

### DrawPalette
- **Signature:** `void DrawPalette(char *name)`
- **Purpose:** Load and install palette from lump
- **Inputs:** name (lump name)
- **Outputs:** None
- **Side effects:** Changes active screen palette
- **Calls:** W_CacheLumpName, memcpy, VL_NormalizePalette, SwitchPalette

### DrawFadeout
- **Signature:** `void DrawFadeout(int time)`
- **Purpose:** Fade screen to black
- **Inputs:** time (fade duration in tics)
- **Outputs:** None
- **Side effects:** Screen fades to black, measures elapsed time
- **Calls:** VL_FadeOut, CalcTics, VL_ClearVideo

### Initialization & Cleanup Functions
- **FirstTimeInitialize, InitializeFilmActors, InitializeEvents, CleanupEvents, CleanupActors:** Manage allocation/deallocation of event and actor arrays; FirstTimeInitialize uses guard flag to run once per session
- **GetNewFilmEvent, GetFreeActor, FreeActor, DeleteEvent:** Pool management for events and actors

**Notes on trivial helpers:** CacheScriptFile, GrabMovieScript, InitializeMovie, CleanupMovie, DumpMovie are straightforward wrappers or initialization routines.

## Control Flow Notes
PlayMovie drives the main frame loop:
1. **Init:** Parse script into event array, initialize actor pool
2. **Loop (filmtics from 0 to movielength by dtime steps):**
   - AddEvents: activate events scheduled at filmtics
   - DrawEvents: render all active actors in layer order
   - UpdateEvents: advance actor timers and positions
   - FlipPage: swap video buffers, CalcTics: measure elapsed time, update dtime
   - Break on ESC or mouse input
3. **Cleanup:** Free all event/actor memory

Rendering order is fixed: palette/fade → background/backgroundscroll → background sprites → backdrop/backdropscroll → foreground sprites. Each layer is rendered atomically (palette/fadeout are exclusive and break early).

## External Dependencies
- `w_wad.h`: WAD resource loader (W_CacheLumpName, W_GetNumForName, W_CacheLumpNum, W_LumpLength)
- `scriplib.h`: Script tokenizer (GetToken, token, script_p, scriptend_p, endofscript, tokenready, ParseNum defined elsewhere)
- `f_scale.h`: Column renderer (R_DrawFilmColumn with register pragma, DrawFilmPost)
- `rt_scale.h`: Fixed-point math and ylookup table
- `z_zone.h`: Memory management (SafeMalloc, SafeFree)
- `rt_vid.h`: Video I/O (VL_NormalizePalette, SwitchPalette, VL_FadeOut, VL_ClearVideo, bufferofs, FlipPage, CalcTics)
- `rt_in.h`: Input polling (IN_ClearKeysDown, IN_GetMouseButtons, LastScan, tics)
- `modexlib.h`: VGA mode (VGAWRITEMAP macro for plane selection)
- `rt_util.h`: Utilities (Error, tics)
- `lumpy.h`: Image structures (patch_t with collumnofs/topoffset, lpic_t with width/height)
