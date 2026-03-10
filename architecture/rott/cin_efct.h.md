# rott/cin_efct.h

## File Purpose
Public header declaring the cinematic effect system interface. Provides factory functions to spawn cinematic events (animations, sprites, backgrounds, palettes) and drawing/update routines to render and advance them during playback. This is the primary API for sequencing cutscenes.

## Core Responsibilities
- Spawn cinematic effect objects (FLICs, sprites, backgrounds, palettes)
- Draw/render active cinematic effects to screen
- Update cinematic effect state each frame (animation frames, sprite positions, scrolling)
- Generic effect dispatch (DrawCinematicEffect, UpdateCinematicEffect) for polymorphic handling
- Precache effect resources before playback
- Manage buffer clearing and screen state
- Performance profiling support

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| flicevent | struct | FLIC animation metadata (name, loop flag, file usage) |
| spriteevent | struct | Sprite animation with position/scale interpolation |
| backevent | struct | Background/backdrop with scrolling offset and duration |
| paletteevent | struct | Palette change event reference |
| enum_eventtype | enum | Event type discriminant (background, sprite, flic, fadeout, etc.) |
| eventtype | struct | Linked-list node for queued cinematic events |
| actortype | struct | Active actor node in playback system |

## Global / File-Static State
None (header only).

## Key Functions / Methods

### SpawnCinematicFlic
- **Signature:** `flicevent * SpawnCinematicFlic ( char * name, boolean loop, boolean usefile )`
- **Purpose:** Factory function to create a FLIC animation event.
- **Inputs:** Animation name, loop flag, usefile flag
- **Outputs/Return:** Pointer to allocated flicevent structure
- **Side effects:** Allocates memory for event; queues into event system
- **Calls:** (Not visible—implementation in .c file)
- **Notes:** "usefile" likely controls whether animation is read from WAD or external file

### SpawnCinematicSprite
- **Signature:** `spriteevent * SpawnCinematicSprite ( char * name, int duration, int numframes, int framedelay, int x, int y, int scale, int endx, int endy, int endscale )`
- **Purpose:** Spawn an animated sprite with position and scale interpolation.
- **Inputs:** Name, animation duration, frame count, frame delay (ticks), start position (x,y,scale), end position (x,y,scale)
- **Outputs/Return:** Pointer to spriteevent
- **Side effects:** Allocates sprite event; queues into system; computes per-frame delta (dx, dy, dscale)
- **Calls:** (Not visible)
- **Notes:** End values suggest linear interpolation over duration; frame state initialized (frame=0, frametime=0)

### SpawnCinematicBack / SpawnCinematicMultiBack
- **Signature:** `backevent * SpawnCinematicBack ( char * name, int duration, int width, int startx, int endx, int yoffset )`; `backevent * SpawnCinematicMultiBack ( char * name, char * name2, int duration, int startx, int endx, int yoffset )`
- **Purpose:** Spawn a scrolling background; MultiBack supports dual-layer parallax.
- **Inputs:** Texture name(s), duration, backdrop width, start/end x offset, y offset
- **Outputs/Return:** Pointer to backevent
- **Side effects:** Allocates background event; computes per-frame scroll delta
- **Notes:** Single/multi variants; width and offsets control scrolling range

### DrawFlic, DrawCinematicBackdrop, DrawCinematicBackground, DrawPalette, DrawCinematicSprite
- **Purpose:** Type-specific rendering functions.
- **Inputs:** Typed event pointers
- **Side effects:** Direct screen/buffer writes; palette updates
- **Notes:** DrawCinematicBackdrop vs. DrawCinematicBackground suggest different rendering modes (noscroll vs. scroll)

### DrawCinematicEffect, UpdateCinematicEffect, PrecacheCinematicEffect
- **Signature:** `boolean DrawCinematicEffect ( enum_eventtype type, void * effect )`; etc.
- **Purpose:** Polymorphic dispatchers; route to type-specific handlers based on enum.
- **Inputs:** Event type discriminant + opaque void pointer
- **Outputs/Return:** Boolean (success/completion status)
- **Notes:** Visitor/dispatcher pattern; allows generic event queue iteration

### UpdateCinematicBack, UpdateCinematicSprite
- **Purpose:** Per-frame state advancement (scrolling, animation frames, interpolation).
- **Outputs/Return:** Boolean (likely true if still active, false if complete)

### DrawClearBuffer, DrawBlankScreen
- **Purpose:** Utility rendering: clear frame buffer and draw blank screen
- **Side effects:** Screen state reset

### DrawPostPic, ProfileDisplay
- **Purpose:** Draw post-process/image resource; performance metrics display
- **Inputs:** DrawPostPic takes lumpnum (WAD resource index)

## Control Flow Notes
Implied cinematic sequence loop:
1. Spawn effects (populate event queue) via Spawn* functions
2. Each frame: UpdateCinematicEffect for all queued events (advance animation/scrolling)
3. Each frame: DrawCinematicEffect for all active events (render to screen)
4. Poll CinematicAbort() (from cin_glob.h); repeat until all events complete or abort
5. Synchronize timing via CinematicDelay() (from cin_glob.h)

## External Dependencies
- `cin_glob.h`: Cinematic timing (CinematicDelay, GetCinematicTime) and abort control
- `cin_def.h`: Type definitions (enums, structs)
- Transitive: `rt_def.h`, `rt_util.h`, `isr.h`, `<time.h>` (from cin_glob.h)
- Graphics/rendering system (not visible in this file; callee of Draw* functions)
- WAD resource system (lumpnum references in DrawPostPic)
