# rott/rt_str.c

## File Purpose
Implements string rendering, text input, and window management for the ROTT game engine. Provides functions for drawing strings in multiple styles (clipped, proportional, intensity-colored), measuring text, handling interactive text input with cursor feedback, and rendering window frames.

## Core Responsibilities
- String and character drawing with various rendering modes (clipped, proportional, intensity-based)
- Text measurement and font metric calculation
- Interactive text input with cursor blinking and editing (line input and password input)
- Window frame rendering using sprite-based borders
- User-facing text printing with alignment and wrapping
- Low-level VGA text mode output for early boot or debugging
- Intensity-based colored font rendering with embedded formatting codes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `font_t` | struct (external) | Standard proportional font with glyph metrics |
| `cfont_t` | struct (external) | Colored/intensity font variant |
| `pic_t` | struct (external) | Picture data for window frame sprites |
| `Rect` | struct (external) | Rectangle defined by upper-left and lower-right corners |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `fontcolor` | int | global | Current color used for intensity string rendering |
| `BKw`, `BKh` | int | static | Background width/height saved for text input cursor region |
| `oldfontcolor` | int | static | Saved font color for highlight restoration |
| `highlight` | boolean | static | Flag tracking active highlight state in intensity strings |
| `disp_offset` | unsigned short | static | Current cursor position in text mode video memory |

## Key Functions / Methods

### VW_DrawClippedString
- **Signature:** `void VW_DrawClippedString(int x, int y, char *string)`
- **Purpose:** Draw a string character-by-character with pixel-level clipping to screen bounds.
- **Inputs:** Starting x,y coordinates; pointer to null-terminated string.
- **Outputs/Return:** None (direct video buffer write).
- **Side effects:** Writes directly to `bufferofs` using `ylookup[]` and `VGAWRITEMAP`.
- **Calls:** None (direct pixel access).
- **Notes:** Skips characters and pixels outside screen bounds. Uses CurrentFont metrics.

### US_LineInput / US_lineinput
- **Signature:** `boolean US_LineInput(int x, int y, char *buf, char *def, boolean escok, int maxchars, int maxwidth, int color)` / `boolean US_lineinput(...)`
- **Purpose:** Blocking text input loop with cursor, editing, and visual feedback. Password variant (US_lineinput) masks input with '*'.
- **Inputs:** Coordinates, default string, char/width limits, escape key permission, color (0=menu buffer, nonzero=screen).
- **Outputs/Return:** true if user pressed Return, false if pressed Escape.
- **Side effects:** Modifies `buf` on success. Updates `px`, `py`, `PrintX`, `PrintY`. Manages `Keyboard[]` array. Calls `VW_UpdateScreen()` or `RefreshMenuBuf()`. Cursor XOR'd to screen repeatedly.
- **Calls:** `IN_ClearKeyboardQueue`, `IN_InputUpdateKeyboard`, `MN_PlayMenuSnd`, `USL_MeasureString`, `USL_XORICursor`, `VWB_Bar`, `EraseMenuBufRegion`, `DrawMenuBufPropString`, `VW_UpdateScreen`, `RefreshMenuBuf`.
- **Notes:** Event loop driven by keyboard scancodes and time-based cursor blink (VBLCOUNTER). Handles arrow keys, home/end, backspace, delete. Validates printable characters and string length.

### US_Print / US_BufPrint
- **Signature:** `void US_Print(char *s)` / `void US_BufPrint(char *s)`
- **Purpose:** Print string to current window or buffer, supporting newlines for line wrapping.
- **Inputs:** Pointer to null-terminated string.
- **Outputs/Return:** None.
- **Side effects:** Updates `PrintX`, `PrintY`, `px`, `py`. Calls the registered print function (USL_DrawString).
- **Calls:** `USL_MeasureString`, `USL_DrawString`.
- **Notes:** Splits on '\n' for multi-line layout. Resets X to WindowX on newline. Uses function pointers set by `US_SetPrintRoutines`.

### VW_DrawPropString / VW_DrawIPropString
- **Signature:** `void VW_DrawPropString(char *string)` / `void VW_DrawIPropString(char *string)`
- **Purpose:** Draw proportional-width string at current (px, py). Standard and intensity variants.
- **Inputs:** Pointer to string.
- **Outputs/Return:** None.
- **Side effects:** Updates `bufferheight`, `bufferwidth`, `px` as it advances. Writes to bufferofs directly.
- **Calls:** None.
- **Notes:** Manages bit-plane mask for VGA's planar architecture. Handles per-pixel width. Updates local copy of `dest` offset as it moves through scan lines.

### VWB_DrawPropString / VWB_DrawIPropString
- **Signature:** `void VWB_DrawPropString(char *string)` / `void VWB_DrawIPropString(char *string)`
- **Purpose:** Wrappers that call the corresponding VW_ function and mark the updated region for screen refresh.
- **Inputs:** Pointer to string.
- **Outputs/Return:** None.
- **Side effects:** Calls `VW_MarkUpdateBlock` to flag screen region for update.
- **Calls:** `VW_DrawPropString` or `VW_DrawIPropString`, `VW_MarkUpdateBlock`.

### US_MeasureStr
- **Signature:** `void US_MeasureStr(int *width, int *height, char *s, ...)`
- **Purpose:** Measure bounding box of formatted string (varargs like printf).
- **Inputs:** Pointers to int width/height (output), format string with arguments.
- **Outputs/Return:** Fills *width and *height.
- **Side effects:** None.
- **Calls:** `vsprintf`, `VWL_MeasureString`.
- **Notes:** Handles newlines and tracks maximum width across all lines.

### US_DrawWindow
- **Signature:** `void US_DrawWindow(int x, int y, int w, int h)`
- **Purpose:** Draw a bordered window frame at tile coordinates (x,y) with width/height in tiles.
- **Inputs:** Top-left tile coordinates, width/height in 8-pixel tiles.
- **Outputs/Return:** None.
- **Side effects:** Sets `WindowX`, `WindowY`, `WindowW`, `WindowH`, `PrintX`, `PrintY`. Loads 9 window sprite pieces from WAD and draws them.
- **Calls:** `W_CacheLumpNum`, `W_GetNumForName`, `VWB_DrawPic`, `US_ClearWindow`.
- **Notes:** Creates 3×3 grid of corner/edge/line sprites. Calls `US_ClearWindow` after drawing frame.

### DrawIString
- **Signature:** `void DrawIString(unsigned short int x, unsigned short int y, char *string, int flags)`
- **Purpose:** Draw string with intensity-based colors and embedded formatting codes (backslash for color, backtick for highlight).
- **Inputs:** Coordinates, string pointer, flags (e.g., PERMANENT_MSG).
- **Outputs/Return:** None.
- **Side effects:** Updates `fontcolor`, `oldfontcolor`, `highlight`. Calls `DrawIntensityChar` repeatedly.
- **Calls:** `DrawIntensityChar`.
- **Notes:** Formatting: `\<hex>` = color next word, `\N<hex>` = set base color, `\O` = restore color, `` ` `` = highlight if fontcolor < 8. Static state tracks highlighting across calls.

### DrawText / myputch / myprintf
- **Signature:** Various (e.g., `void DrawText(int x, int y, int ch, int fg, int bg)`, `int myprintf(char *fmt, ...)`)
- **Purpose:** Low-level text mode (0xB0000) output for early boot or debugging.
- **Inputs:** Coordinates, character, color attributes, or printf-style format string.
- **Outputs/Return:** Character count for printf variant.
- **Side effects:** Direct writes to text video memory at 0xB0000. `myputch` auto-scrolls if offset exceeds 160×24. `myprintf` calls `Debug()` if MONOPRESENT is false.
- **Calls:** `myputch`, `printstring`, `printnum`, `printunsigned`, `Debug`.
- **Notes:** Used before full video subsystem is initialized. Implements subset of printf (d, l, s, u, x).

## Control Flow Notes
- **Text input loop:** `US_LineInput` and `US_lineinput` are blocking event loops that poll keyboard and update cursor blink based on timer (`ticcount`). They repeatedly draw/erase the cursor using XOR.
- **String drawing:** Most draw functions write directly to `bufferofs` at calculated offsets, relying on CurrentFont or IFont metrics.
- **Window management:** Window drawing is often preceded by `US_ClearWindow` to set background color and reset print position.
- **Intensity strings:** `DrawIString` maintains static state across calls to handle multi-word highlighting and color context.
- **Text mode functions:** Used during early initialization before mode-X graphics are active, or as a fallback debug channel.

## External Dependencies
- **Standard library:** `stdlib.h`, `stdio.h`, `stdarg.h`, `string.h`, `ctype.h`
- **Engine core:** `rt_def.h` (constants, types), `rt_menu.h` (menu structures), `rt_in.h` (keyboard input), `rt_vid.h` (video output)
- **Graphics:** `lumpy.h` (font_t, pic_t structures), `modexlib.h` (VGA primitives)
- **Memory & assets:** `w_wad.h` (W_CacheLumpNum), `z_zone.h` (zone allocator)
- **Utilities & subsystems:** `rt_util.h`, `rt_build.h` (menu buffer), `rt_sound.h` (MN_PlayMenuSnd), `isr.h`, `rt_main.h`, `memcheck.h`
- **Defined elsewhere:** `CurrentFont`, `IFont`, `bufferofs`, `ylookup[]`, `linewidth`, `egacolor[]`, `intensitytable`, `Keyboard[]`, `LastScan`, `ticcount`, `VBLCOUNTER`, `PrintX`, `PrintY`, `WindowX/W/Y/H`, `px`, `py`, `bufferheight`, `bufferwidth`, `MONOPRESENT`
