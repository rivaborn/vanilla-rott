# rott/rt_film.h — Enhanced Analysis

## Architectural Role
This header provides a thin public interface to the cinematic/demo sequence playback subsystem. In the engine's execution model, `PlayMovie()` acts as a blocking interrupt in the normal game loop—typically called from menus or initialization routines to display pre-rendered cutscenes or demos. The `dc_ycenter` global bridges the film renderer to the display system, allowing vertical centering independent of screen resolution or configuration.

## Key Cross-References
### Incoming (who depends on this file)
Cross-reference data provided is limited to function maps; specific callers of `PlayMovie` not visible in the excerpt. Likely callers would be:
- Menu system (likely `rt_menu.c` or similar)—for demo playback in main menu
- Game initialization/flow control—for intro sequences
- Possibly script/cinematic engine code (given `cin_*.c` files in the codebase)

### Outgoing (what this file depends on)
- `rt_film.c` — implementation partner (private details not visible in this header)
- Display/rendering system — for screen output, palette management
- File I/O subsystem — to load movie files
- Cinematic system — cross-references show `cin_actr.c`, `cin_evnt.c`, `cin_glob.c`, `cin_util.c`, suggesting possible integration with a broader cinematic scripting engine

## Design Patterns & Rationale
**Facade Pattern**: A single public function (`PlayMovie`) hides all implementation complexity—decompression, frame timing, palette management, input handling, and cleanup.

**Global State for Output Parameters**: Rather than returning positioning data, `dc_ycenter` is exported as a mutable global. This is typical of 1990s engine design, particularly for rendering parameters that must be accessible across module boundaries without indirect function calls. The alternative (returning a struct or using output parameters) would require more coupling.

**Blocking Execution Model**: `PlayMovie` appears to block until completion rather than returning control immediately. This is characteristic of engines where cinematics are "state transitions" rather than concurrent systems—the player surrenders control to the movie renderer until playback ends.

## Data Flow Through This File
```
Caller (menu/init) 
  → PlayMovie(filename: string)
    → [rt_film.c] loads file, decompresses frames
    → Sets dc_ycenter (read by renderer)
    → Display loop processes frames, checks for abort conditions
    → Returns to caller when playback ends
```

The exported `dc_ycenter` is read—not written—by the caller's rendering code; the film module controls it.

## Learning Notes
- **Era-typical cinematic integration**: This engine treats cinematics as a separate subsystem that temporarily hijacks the main loop, rather than playing them within a scene graph or entity system (as modern engines do).
- **Minimal abstraction**: Only the bare API is exposed; all state management, timing, and event handling remain private to `rt_film.c`. This reduces coupling but also limits the caller's control.
- **String-based resource loading**: Movie filename as a raw `char*` with no documented format or path constraints suggests a simple, convention-based system.
- **Module isolation**: The header makes no references to other engine systems (actors, events, network, sound), implying cinematics are either self-contained or communicate only through global state.

## Potential Issues
- **No error handling**: `PlayMovie` returns `void`; no way to signal playback failure (file not found, corruption, unsupported format).
- **Unvalidated input**: The filename pointer is unchecked; no null-check, length validation, or path sanitization visible from the signature.
- **Blocking design fragility**: If the caller needs responsiveness during playback (e.g., to show a progress bar or handle network packets), the blocking model provides no hook. Modern engines would use asynchronous/frame-based playback.
- **Global positioning state**: `dc_ycenter` being a mutable global invites accidental misuse; thread-unsafe by design, and its lifecycle (when is it valid?) is unclear from the header alone.
