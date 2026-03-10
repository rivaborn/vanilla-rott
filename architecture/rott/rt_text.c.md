# rott/rt_text.c

## File Purpose
Implements custom text markup rendering and page layout for the ROTT game engine. Parses a domain-specific markup language (with commands like `^C`, `^G`, `^P`) to render formatted text, graphics, and paginated articles with dynamic margin management.

## Core Responsibilities
- Parse and execute custom text formatting commands (`^C` color, `^G` graphic, `^P` page break, `^L` locate, `^T` timed graphic, `^E` end, `^B` bar)
- Implement word-wrapping text layout with per-row margin constraints
- Manage graphics insertion and adjust text margins around graphics
- Navigate between pages in multi-page articles
- Cache graphics resources before rendering
- Handle keyboard input for page navigation (up/down/escape)

## Key Types / Data Structures
None (uses scalar types and external `pic_t` structure).

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `pagenum`, `numpages` | int | static | Current page number and total page count |
| `leftmargin`, `rightmargin` | unsigned array[TEXTROWS] | static | Per-row text margin constraints |
| `text` | char* | static | Pointer into current text stream being parsed |
| `rowon` | unsigned | static | Current row (line) being laid out |
| `picx`, `picy`, `picnum`, `picdelay` | int | static | Parsed picture positioning and parameters |
| `layoutdone` | boolean | static | Flag: current page layout complete |
| `GFX_STRT` | int | static | Starting lump index for graphics |
| `FONTCOLOR` | int | static | Current text color (palette index) |
| `Pic` | pic_t* | static | Pointer to currently cached picture |
| `str`, `str2` | char array | static | Temporary string buffers |
| `px`, `py` | (inferred) | static | Current pixel-position for text rendering |

## Key Functions / Methods

### ParseNumber
- **Signature:** `int ParseNumber(void)`
- **Purpose:** Extract a decimal number from the current position in the text stream.
- **Inputs:** Global `text` pointer (modified in-place).
- **Outputs/Return:** Parsed integer value.
- **Side effects:** Advances `text` past the number.
- **Calls:** `atoi()`
- **Notes:** Scans non-digits first, then copies digits into temporary buffer.

### ParsePicCommand
- **Signature:** `void ParsePicCommand(void)`
- **Purpose:** Parse picture command parameters: y, x, picture number.
- **Inputs:** Global `text` pointer positioned after `^G`.
- **Outputs/Return:** Sets global `picy`, `picx`, `picnum`.
- **Side effects:** Advances `text` to start of next line.
- **Calls:** `ParseNumber()`, `RipToEOL()`
- **Notes:** Command format: `^Gyyy,xxx,ppp[enter]`.

### ParseTimedCommand
- **Signature:** `void ParseTimedCommand(void)`
- **Purpose:** Parse timed graphic command: y, x, picture number, delay ticks.
- **Inputs:** Global `text` pointer positioned after `^T`.
- **Outputs/Return:** Sets global `picy`, `picx`, `picnum`, `picdelay`.
- **Side effects:** Advances `text` to next line.
- **Calls:** `ParseNumber()`, `RipToEOL()`
- **Notes:** Command format: `^Tyyy,xxx,ppp,ttt[enter]`.

### TimedPicCommand
- **Signature:** `void TimedPicCommand(void)`
- **Purpose:** Wait for specified time, then draw a picture to screen.
- **Inputs:** Global `text` pointer (after parsing).
- **Outputs/Return:** None.
- **Side effects:** Updates screen, waits for `picdelay` tics, caches and draws graphic.
- **Calls:** `ParseTimedCommand()`, `VW_UpdateScreen()`, `W_CacheLumpNum()`, `VWB_DrawPic()`
- **Notes:** Busy-waits on global `ticcount`. Uses global `GFX_STRT` and `Pic`.

### HandleCommand
- **Signature:** `void HandleCommand(void)`
- **Purpose:** Process control sequences (^X where X is the command character).
- **Inputs:** Global `text` pointing just after the `^`.
- **Outputs/Return:** None.
- **Side effects:** Modifies global state (`FONTCOLOR`, margins, `px`, `py`, `layoutdone`, cached `Pic`). Draws graphics and rectangles.
- **Calls:** `toupper()`, `ParseNumber()`, `RipToEOL()`, `TimedPicCommand()`, `ParsePicCommand()`, `W_CacheLumpNum()`, `VWB_DrawPic()`, `VWB_Bar()`
- **Notes:** 
  - Supports: `C` (color), `;` (comment), `P`/`E` (end), `B` (bar), `>` (center), `L` (locate), `T` (timed), `G` (graphic).
  - Case-insensitive.
  - Graphics automatically adjust left/right margins on subsequent rows.

### HandleWord
- **Signature:** `void HandleWord(void)`
- **Purpose:** Render a single word with automatic line wrapping.
- **Inputs:** Global `text` pointing at first character of word.
- **Outputs/Return:** None.
- **Side effects:** Advances `text`, updates `px`, calls `VWB_DrawPropString()`, may call `NewLine()` if word doesn't fit.
- **Calls:** `VW_MeasurePropString()`, `NewLine()`, `VWB_DrawPropString()`
- **Notes:** Skips trailing spaces; handles word-length limit (80 chars).

### NewLine
- **Signature:** `void NewLine(void)`
- **Purpose:** Advance layout to next row; handle page overflow.
- **Inputs:** None (uses global state).
- **Outputs/Return:** None.
- **Side effects:** Increments `rowon`, updates `py`, sets `layoutdone` if overflow. Scans for next page break on overflow.
- **Calls:** `toupper()`
- **Notes:** If `rowon >= TEXTROWS`, sets `layoutdone = true` and seeks to next `^E` or `^P`.

### PageLayout
- **Signature:** `void PageLayout(boolean shownumber)`
- **Purpose:** Main layout engine: clear screen, initialize margins, parse and render text stream to current page.
- **Inputs:** `shownumber` – whether to display page number on screen.
- **Outputs/Return:** None.
- **Side effects:** Clears screen, resets margins and position, processes entire text stream until `^P` or `^E`, draws all graphics and text, increments `pagenum`. Caches and renders page number string if requested.
- **Calls:** `VWB_Bar()`, `HandleCommand()`, `HandleCtrls()`, `HandleWord()`, `VWB_DrawPropString()`, `strcpy()`, `itoa()`, `strcat()`
- **Notes:** Tabs are converted to next 8-pixel boundary. Assumes text starts with `^P` (page-start marker).

### CacheLayoutGraphics
- **Signature:** `void CacheLayoutGraphics(void)`
- **Purpose:** Scan entire layout file to count pages and identify all graphics, preparing for rendering.
- **Inputs:** Global `text` pointer (set to start of article).
- **Outputs/Return:** None.
- **Side effects:** Counts `^P` commands into `numpages`. Resets `text` to start on return. Searches for terminating `^E`.
- **Calls:** `toupper()`, `ParsePicCommand()`, `ParseTimedCommand()`
- **Notes:** Commented-out calls to `CA_MarkGrChunk()` suggest graphics marking was disabled. Fails with error if `^E` not found within 30KB.

### ShowArticle
- **Signature:** `void ShowArticle(char *article)`
- **Purpose:** Display a multi-page article with keyboard navigation.
- **Inputs:** `article` – pointer to text buffer (containing markup).
- **Outputs/Return:** None.
- **Side effects:** Clears screen, caches graphics, loops rendering pages and handling input (up/down/escape), calls `MenuFadeIn()` on first page, clears keyboard state on exit.
- **Calls:** `W_GetNumForName()`, `VWB_Bar()`, `CacheLayoutGraphics()`, `PageLayout()`, `VW_UpdateScreen()`, `MenuFadeIn()`, `BackPage()`, `IN_ClearKeysDown()`
- **Notes:** Navigation: up/left/pgup go back two pages; down/right/pgdn/enter go forward. Escape exits.

## Control Flow Notes
**Init/Frame/Render:**
- `ShowArticle()` is the game-facing entry point, typically called from menu/help systems.
- On entry, `CacheLayoutGraphics()` pre-processes the entire article to measure pages.
- Main loop: `PageLayout()` renders one page per iteration, `VW_UpdateScreen()` displays, then waits for keyboard input.
- Navigation updates state (page number, text pointer) and loops.
- On page change, `PageLayout()` re-parses the markup stream from the current position.

**Shutdown:**
- User presses Escape to exit the loop; `IN_ClearKeysDown()` clears pending input.

## External Dependencies
- **Graphics/Video:** `W_CacheLumpNum()`, `W_GetNumForName()`, `VW_UpdateScreen()`, `VWB_DrawPic()`, `VWB_Bar()`, `VWB_DrawPropString()`, `VW_MeasurePropString()`, `MenuFadeIn()`
- **Input:** `IN_ClearKeysDown()`, global `LastScan` (keyboard scan code), global `ticcount` (frame counter)
- **Scan codes referenced:** `sc_UpArrow`, `sc_PgUp`, `sc_LeftArrow`, `sc_DownArrow`, `sc_PgDn`, `sc_RightArrow`, `sc_Enter`, `sc_Escape`
- **Standard library:** `stdlib.h` (atoi), `ctype.h` (toupper, isdigit implied), `string.h` (strcpy, strcat)
- **Project headers:** `RT_DEF.H` (types, constants), `memcheck.h` (memory debugging)
