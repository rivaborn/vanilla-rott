# rott/myprint.h — Enhanced Analysis

## Architectural Role
This module provides the **text rendering subsystem** for Rise of the Triad's UI and debugging layers. It abstracts a DOS-era 16-color text mode, bridging high-level callers (menu system, debug overlay, HUD) and the underlying video buffer. Functions range from low-level cursor/character primitives to high-level formatted output (`myprintf`) and decorated UI elements (frames, boxes).

## Key Cross-References

### Incoming (who depends on this file)
The cross-reference index provided does not include callers of `myprint.h` functions. However, based on function signatures and idiom:
- **Menu/UI subsystem** (likely `rt_menu.c`, `rt_menu.h`) — calls `DrawText`, `TextFrame`, `TextBox` for dialog boxes and menus
- **Debug overlay** (likely `rt_debug.c`) — calls `myprintf`, `printstring`, `printnum` for debug output
- **HUD/status display** — calls `myprintf` and `DrawText` for player info, weapon status
- Likely called from main render loop or UI event handling

### Outgoing (what this file depends on)
Header only; implementation (`myprint.c`, not shown) would likely depend on:
- **Video buffer abstraction** — writes to memory-mapped screen buffer or console interface (underlying screen driver unknown from header)
- **Character/color encoding** — the `COLORS` enum maps to DOS color indices (0-15); character codes use standard DOS/ASCII

## Design Patterns & Rationale

**Immediate-Mode API**: No retained state; functions operate directly on screen coordinates. This is characteristic of DOS-era game engines and contrasts sharply with modern retained-mode (scene graph) or batch-rendering approaches.

**Layered Abstraction**:
- **Layer 1** (lowest): `myputch`, `mysetxy` — single-character, single-position primitives
- **Layer 2** (mid): `printstring`, `printnum`, `printunsigned` — format strings/numbers at current cursor
- **Layer 3** (high): `myprintf`, `DrawText`, `TextBox`, `TextFrame` — coordinate-based or structured output

**Fixed Color Palette**: The 16-color enum reflects VGA text mode constraints (typical DOS limitation). No color blending, no alpha, no dynamic palette—a hard constraint of the hardware.

---

## Data Flow Through This File

```
User Input (menu selection, debug event, HUD update)
    ↓
High-level UI code calls: DrawText(), TextFrame(), myprintf()
    ↓
String/number formatting (myprintf delegates to printstring, printnum, printunsigned)
    ↓
Cursor positioning (mysetxy) and character output (myputch)
    ↓
Screen buffer (implementation detail; likely DOS console or memory-mapped VRAM)
```

No state is retained across calls; each operation is independent.

---

## Learning Notes

**Era-Specific Idioms**:
- **Variadic printf**: `myprintf` mirrors C's `printf`, suggesting this codebase adapted printf-style conventions for game output (common in 90s game engines before modern logging frameworks)
- **Color as integer enum**: No color objects or abstraction; colors are plain `int` indices—typical of DOS/VGA constraints
- **Direct screen access**: No framebuffer abstraction; assumes synchronous writes to a single, always-visible screen buffer

**Modern Contrast**: A modern engine would use:
- GPU-based text rendering (shaders, texture atlases)
- Batched draws with deferred rendering
- Retained scene graph (UI hierarchy)
- Vector graphics, not DOS text mode

**Architectural Position**: This is part of the **I/O subsystem**, parallel to audio (`audiolib/`) and input. It's **not** part of the 3D rendering pipeline; it's the 2D overlay for menus, HUD, and debug info.

---

## Potential Issues

- **No bounds checking in signatures**: Functions like `DrawText(int x, int y, ...)` have no documented clipping behavior. If caller passes out-of-bounds coordinates, behavior depends on implementation (likely undefined or buffer overflow risk).
- **Cursor state hidden**: `mysetxy` and `myputch` imply a persistent cursor position, but this state is not visible in the header. Callers cannot query the current cursor position—risk of state desync if `mysetxy` is called inconsistently.
- **No error handling**: All functions return `void` or `int` (presumed character count), with no error codes. Format errors in `myprintf` would silently fail.

---

**Note**: The provided cross-reference context does not include calls *to* `myprint.h` functions, only calls *within* other subsystems. A complete call graph would require scanning `.c` files for `#include "myprint.h"` and analyzing callers.
