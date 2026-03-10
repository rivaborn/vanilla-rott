Looking at the first-pass analysis and the cross-reference context provided, I'll write the enhanced second-pass analysis:

# rott/rt_text.c — Enhanced Analysis

## Architectural Role
This file implements the **game's article/help system UI**, serving as a custom markup renderer layer between the menu system and the graphics subsystem. It specializes in **page-based text layout with embedded graphics**, designed for in-game help screens and story interludes (like game cinematics or briefings). The file bridges the **menu/input layer** (which calls `ShowArticle()`) and the **video/graphics layer** (VWB_* functions), handling all parsing, layout logic, and keyboard navigation autonomously.

## Key Cross-References

### Incoming (who depends on this file)
- **Menu system** (referenced in first-pass) calls `ShowArticle()` for help/article display
- **Main game loop** provides `ticcount` global (frame counter for timed graphics)
- **Input system** provides `LastScan` global (keyboard scan codes) and `IN_ClearKeysDown()` cleanup
- `CacheLayoutGraphics` is listed in the cross-reference map as defined here (caller unknown from excerpt, but logically called by `ShowArticle()` internally)

### Outgoing (what this file depends on)
- **Graphics subsystem**: `W_CacheLumpNum()` (resource caching), `VWB_DrawPic()` (sprite drawing), `VWB_Bar()` (rectangle fill), `VWB_DrawPropString()` (text rendering), `VW_MeasurePropString()` (text measurement), `VW_UpdateScreen()` (framebuffer swap)
- **Resource system**: `W_GetNumForName()` (lump lookup by name)
- **Menu system**: `MenuFadeIn()` (screen fade-in effect on first page)
- **Input system**: `IN_ClearKeysDown()` (flush pending input on exit)
- **Globals**: `ticcount` (frame counter), `LastScan` (current key press)

## Design Patterns & Rationale

1. **Custom Domain-Specific Language (DSL)**: The `^X` command syntax (e.g., `^C`, `^G`, `^P`) is a lightweight markup approach optimized for 1990s constraints—avoids runtime HTML/Markdown parsing overhead. Format: `^C<hex>` (color), `^Gyyy,xxx,ppp` (graphic), `^P` (page break), `^L` (locate), `^T` (timed graphic).

2. **Stateful Parser Pattern**: Global `text` pointer acts as implicit instruction pointer through the markup stream. State is split across globals (`px`, `py`, `rowon`, `leftmargin[]`, `rightmargin[]`, `FONTCOLOR`, `layoutdone`), making the layout engine a **state machine** rather than a functional pipeline. This was likely necessary for incremental rendering on low-memory systems.

3. **Pre-scan then Render**: `CacheLayoutGraphics()` makes a full pass to count pages and collect graphics metadata before any rendering. This enables page-count display and ensures graphics are cached before drawing, avoiding mid-render stalls.

4. **Per-row Margin Management**: Each text row tracks left and right margins separately (`leftmargin[TEXTROWS]`, `rightmargin[TEXTROWS]`), enabling sophisticated **text wrapping around embedded graphics** without explicit margin-stack or scene-graph structure.

5. **Busy-Wait Synchronization**: `TimedPicCommand()` uses `while(ticcount < delay+picdelay)` instead of event-driven timers—reflects era and hardware constraints (no OS timers, interrupt-driven `ticcount` was the only reliable clock).

## Data Flow Through This File

```
ShowArticle(article_ptr)
    ├─ W_GetNumForName() → fetch lump index for graphics
    ├─ CacheLayoutGraphics() → scan entire text, count pages, identify graphics
    │  └─ Advances text pointer through all ^P, ^G, ^T commands
    └─ Main loop (pagenum=1; pagenum<=numpages):
        ├─ PageLayout(shownumber)
        │   ├─ Clear screen (VWB_Bar)
        │   ├─ Initialize margins (left/right[])
        │   ├─ Parse text stream line-by-line:
        │   │   ├─ HandleCommand() → ^C/^G/^P/^L/^T/^B/^E
        │   │   │   ├─ W_CacheLumpNum() + VWB_DrawPic() for graphics
        │   │   │   └─ Updates margins[] if graphic inserted
        │   │   ├─ HandleCtrls() → newline handling
        │   │   └─ HandleWord() → measure + wrap + draw via VWB_DrawPropString()
        │   └─ Render page number if requested
        ├─ VW_UpdateScreen() → flip framebuffer
        └─ Input loop (check LastScan for page nav):
            ├─ Up/PgUp/Left → BackPage() (pagenum -= 2)
            ├─ Down/PgDn/Enter/Right → pagenum++
            └─ Escape → exit loop
```

**Key state transitions:**
- `text` pointer advances through markup stream during `PageLayout()`
- `rowon` (current row) resets per page, increments as lines are added
- `layoutdone` flag stops layout when `^P` or `^E` encountered
- Margins are per-page fresh state (reset on each `PageLayout()` call)

## Learning Notes

1. **Pre-GPU Text Rendering**: VWB_DrawPropString() is a CPU-side proportional font renderer, likely bitblitting glyphs from a font atlas into the framebuffer. Contrasts sharply with modern GPU-based text (OpenGL/Vulkan shaders, font atlases in GPU memory).

2. **Fixed-Layout Assumption**: Hardcoded 320×200 screen, 10-pixel font height, TEXTROWS = (200-48)/10 = 15 rows. Modern engines would parameterize these; this code assumes one target resolution.

3. **No Object Abstraction**: No `struct ArticleState` or similar—state is scattered across file-scope globals. This was a common 1990s practice but makes concurrent article displays impossible and complicates debugging.

4. **Implicit Resource Lifetime**: Graphics loaded on-demand via `W_CacheLumpNum(..., PU_CACHE)`. The `PU_CACHE` tag hints at a **purge-able cache** system (likely LRU or age-based), where resources can be freed if memory is needed. No explicit cleanup in this file—assumes the cache manager handles deallocation.

5. **Manual Memory Layout**: The text wrapping logic manually calculates pixel widths and margin constraints—no automatic layout system (like CSS Flexbox). Developers had to carefully craft markup to avoid overflowing or overlapping text and graphics.

6. **Era-Specific Input Handling**: Polling `LastScan` global rather than event queues. Common on DOS/early Windows; modern engines use event callbacks or input managers.

## Potential Issues

1. **Busy-Wait CPU Spin** (rt_text.c:155–161): `TimedPicCommand()` spins on `ticcount` with no yield or sleep. On modern systems, this would waste CPU; acceptable on 1990s hardware where CPU-bound synchronization was normal.

2. **Array Bounds Vulnerable** (rt_text.c:328, 355): `leftmargin[rowon]` and `rightmargin[rowon]` accessed without bounds checks. If graphics are positioned such that `rowon` exceeds `TEXTROWS`, undefined behavior. Defensive check at NewLine (line 286) only catches overflow during new line, not retroactive access.

3. **Markup Parsing Fragility** (rt_text.c:89–115): `ParseNumber()` scans from `*text` forward until a digit is found, then copies digits until non-digit. If markup is malformed (missing numbers after `^G`), behavior is undefined—could scan far into unrelated text, causing wrong graphic coordinates.

4. **Single-Instance Global State** (rt_text.c:69–85): File-scope globals mean only one article can be displayed at a time. Attempting nested or concurrent calls (e.g., help menu inside a help article) would corrupt state. No re-entrancy guard.

5. **Hardcoded Limits** (rt_text.c:47–53): `WORDLIMIT=80`, `TEXTROWS=15`. Exceeding word length triggers `Error()` (fatal crash). Long URLs or technical terms could crash the help system.

6. **Unvalidated Command Stream** (rt_text.c:130–160): If a custom article lump is corrupt or ends before a terminating `^E`, `CacheLayoutGraphics()` scans up to 30KB with no safety stop. Could read uninitialized memory if the lump is truncated.

7. **No Error Recovery**: Commands like `^C` and `^L` parse numbers blindly. If the format is wrong (e.g., `^L0,0` without coordinates), `ParseNumber()` may loop indefinitely or read garbage from adjacent memory.
