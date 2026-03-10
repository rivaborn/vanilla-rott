# rott/rt_film.c — Enhanced Analysis

## Architectural Role

**rt_film.c** is the playback and rendering engine for Rise of the Triad's cinematic cutscenes, implementing a frame-based timeline system for scripted sequences. It sits at the intersection of asset loading (WAD), scripting (script parsing), and real-time rendering (VGA video). While the broader `cin_*.c` subsystem handles high-level event creation and cinematic state, **rt_film.c** is responsible for the low-level frame loop: parsing script timelines into discrete events, activating them when scheduled, rendering them in correct z-order, and advancing their state each frame.

## Key Cross-References

### Incoming (who depends on this file)
- **cin_main.c** → calls `PlayMovie(name)` as the primary entry point (matches `CacheScriptFile` reference in cin_main.c)
- **rt_menu.h** → cinematic cutscenes likely triggered from menu system (via PlayMovie)
- **rt_game.c** → likely invokes cinematics at level boundaries (story progression)
- **_rt_film.h** (companion header) → provides public interface to rt_film.c

### Outgoing (what this file depends on)
- **w_wad.h** → loads sprite/background lumps via W_CacheLumpName (textures, backdrops, palettes)
- **scriplib.h** → parses `.ms` movie scripts via GetToken, token, script_p/scriptend_p globals
- **rt_in.h** → polls input (ESC key, mouse buttons) to allow canceling cinematics
- **rt_vid.h** → video output via bufferofs, FlipPage, palette switching (SwitchPalette, VL_FadeOut)
- **rt_scale.h** → ylookup table and fixed-point math for sprite scaling
- **f_scale.h** → column renderer (R_DrawFilmColumn) for scaled sprite columns
- **z_zone.h** → memory allocation (SafeMalloc/SafeFree for events and actors)
- **modexlib.h** → VGAWRITEMAP for VGA planar rendering

## Design Patterns & Rationale

### Timeline / Frame-Based Animation
**rt_film.c** implements a classic timeline pattern:
1. **Parse phase**: Script defines discrete events at absolute tic times (e.g., "at tic 100, show sprite X")
2. **Activate phase** (AddEvents): When playback reaches a scheduled time, spawn an actor (active instance)
3. **Render phase** (DrawEvents): Draw all active actors in fixed layer order
4. **Update phase** (UpdateEvents): Advance all actor timers and positions, free expired actors

This structure decouples script parsing from runtime, allowing flexible event composition without frame-by-frame scripting.

### Layer-Based Rendering
The fixed rendering order (palette/fade → background → background sprites → backdrop → foreground sprites) achieves z-depth without explicit sorting. **Why this works**: Cinema events are pre-authored with intended layering; the engine enforces it rather than computing it. This is efficient (no sort overhead) but requires script discipline.

### Actor Pooling
Static arrays (`actors[MAXFILMACTORS]`, `events[MAXEVENTS]`) avoid dynamic allocation per frame. The `lastfilmactor` index tracks the highest-allocated slot to avoid unnecessary loop iterations. **Tradeoff**: Fixed max actors (no overflow gracefully), but predictable memory and no fragmentation.

### One-Time Events (Palette, Fadeout)
Events like palette changes and fades set `tics=-1` (infinite duration), preventing them from being freed by timeout. Instead, `DeleteEvent` removes them when a new exclusive event of the same type arrives. **Rationale**: Palette/fade are idempotent; no point in repeating them.

## Data Flow Through This File

**Film Playback Timeline:**
```
Script File (lumps with .ms data)
    ↓
ParseMovieScript() [tokenizes with scriplib globals]
    ↓
events[] array (static, indexed by eventindex)
    ↓
PlayMovie() Frame Loop:
  1. AddEvents(): filmtics → check events[currentevent].time → allocate actors[]
  2. DrawEvents(): iterate actors[], render via DrawSprite/DrawBackground/etc. in z-order
  3. UpdateEvents(): advance actor state (positions, scales, timers)
  4. FlipPage() / CalcTics(): swap video buffer, measure elapsed real time (dtime)
  5. Loop until filmtics >= movielength or user presses ESC
    ↓
CleanupMovie() → free events[] and actors[]
```

**Key State Variables:**
- `filmtics` (int): Current playback position in game tics
- `dtime` (int): Elapsed tics since last frame (drives UpdateEvents loop)
- `currentevent` (int): Index of next event in event queue to activate
- `eventindex` (int): Count of parsed events (also free slot index during parsing)
- `filmbuffer` (byte*): Points to current VGA video buffer (reassigned per frame from bufferofs)

## Learning Notes

### Idiomatic to Early 1990s Game Engines
1. **Timeline scripting over code**: Events are data, not logic. Cinematics are data-driven (script files), not programmed. Compare modern engines (Unity, Unreal) which use timeline editors; ROTT used text scripts.
2. **Fixed-point math for scaling**: `dc_invscale` and `dc_iscale` (from rt_scale.h) represent 1/scale ratios as fixed-point integers. Avoids floating-point overhead on 486-era CPUs. Modern engines use float freely.
3. **Planar VGA rendering**: VGAWRITEMAP switches hardware planes per-column for 4-plane rendering. Necessary on VGA hardware; modern engines use flat linear framebuffers.
4. **Column-major sprite format**: Sprites store offsets per-column (patch_t), allowing efficient vertical scaling of single columns. Contrast with modern row-major PNG/texture formats.
5. **Screen-space clipping**: DrawSprite clips to [0,320)×[0,200) bounds inline, not via scissor test. No GPU, so CPU must cull.

### Modern Engine Differences
- **Scene graphs** (ROTT: none; actors are flat list) → hierarchical transforms
- **Shaders** (ROTT: hardcoded column renderer) → programmable pipelines
- **Tween libraries** (ROTT: manual dx/dy/dscale fields) → animation curves (Bezier, etc.)
- **Event systems** (ROTT: monolithic PlayMovie loop) → decoupled event listeners

### Game Engine Concepts
- **Timeline/Sequencer pattern**: Events with scheduled activation times (used in Unreal's Matinee, Unity Timeline)
- **Actor/Sprite pooling**: Reuse allocations to reduce GC pressure (still relevant in modern engines)
- **Layer-based rendering**: Early 2D games avoided sorting by assigning layers; modern engines compute z-order dynamically

## Potential Issues

1. **No recursion guard in AddEvents**: If two events are scheduled at the same tic with conflicting types (background + palette), `DeleteEvent` is called during `AddEvents` loop. Works because `actors[]` is modified in-place, but fragile if logic changes.

2. **Fixed array bounds**: No graceful overflow if eventindex > MAXEVENTS or lastfilmactor > MAXFILMACTORS. Error() terminates the program. Modern engines would queue or warn.

3. **Tight coupling to VGA hardware**: VGAWRITEMAP and ylookup are VGA-specific. Porting to modern graphics (OpenGL/Direct3D) requires rewriting DrawSprite, DrawFilmPost, ScaleFilmPost entirely.

4. **Memory leak risk in ParseMovieScript**: If Error() is called mid-parse, allocated events[] are not freed. Relies on game exit cleanup rather than exception safety.

5. **No async loading**: Lumps are cached on-demand (W_CacheLumpName inside DrawEvents). Could cause frame stalls if lumps are large or disk I/O is slow.
