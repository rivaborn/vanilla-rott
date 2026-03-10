# rott/_st_menu.h

## File Purpose
Private header for the menu subsystem (st_menu.c). Defines constants, data structures, and function prototypes for rendering and handling the game's UI menus, including main menu, modem setup, music/sound configuration, and serial port selection.

## Core Responsibilities
- Define menu layout constants (colors, screen positions, dimensions)
- Declare the `EditList` structure for storing phonebook/configuration entries
- Declare all menu rendering functions (draw, erase, position cursor, display info)
- Declare all menu handler functions (input, selection, validation)
- Define static menu data structures for main menu, sub-menus, and configuration dialogs
- Provide menu-specific handlers for modular features (modem, serial, music, FX, etc.)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `EditList` | struct | Holds a numbered entry (phone number / name pair) for phonebook or configuration list (max 15-char number, 40-char name) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `STMainItems` | `CP_iteminfo` | static | Main menu layout (position, item count, spacing) |
| `STMainMenu` | `CP_itemtype[]` | static | Main menu items (Serial, Modem, Music/Sound, Stuff, Quit) |
| `MusSndItems` | `CP_iteminfo` | static | Music/Sound submenu layout |
| `MusSndMenu` | `CP_itemtype[]` | static | Music/Sound items (Music, SFX, volumes, Escape) |
| `ModemItems` | `CP_iteminfo` | static | Modem submenu layout |
| `ModemMenu` | `CP_itemtype[]` | static | Modem items (Quick dial, Manual, Phonebook, Setup, Escape) |
| `StuffItems` | `CP_iteminfo` | static | Stuff submenu layout |
| `StuffMenu` | `CP_itemtype[]` | static | Stuff items (Reset sounds, Graphics, Levels, Escape) |
| `MusicItems`, `MusicMenu` | static arrays | static | Music device selection (GenMIDI, Canvas, WBlast, SBlast, USound, Adlib, PAS, None) |
| `FXItems`, `FXMenu` | static arrays | static | Sound FX device selection (same devices as music) |
| `MPItems`, `XItems1–3`, `SerialItems`, `PortItems`, `EditItems`, `ModemSetItems` | static arrays | static | Configuration menus for ports, serial settings, FX modes, baud rates, init/answer strings |

## Key Functions / Methods

### MN_DrawMainMenu
- **Purpose:** Render the main menu to screen
- **Calls (visible):** Not fully inferable; likely calls lower-level drawing functions

### MN_HandleMenu
- **Signature:** `int MN_HandleMenu(CP_iteminfo *item_i, CP_itemtype *items)`
- **Purpose:** Process user input for a generic menu (keyboard navigation, selection)
- **Inputs:** Menu metadata (`item_i`) and item array (`items`)
- **Outputs/Return:** Selected item index or -1 if escape/cancel

### MN_DrawCurosr / MN_EraseCursor
- **Purpose:** Render or remove the selection cursor at a menu position
- **Inputs:** Menu info, current/new position, base Y coordinate
- **Notes:** Function name has a typo ("Curosr" instead of "Cursor")

### MN_DrawMenu
- **Signature:** `void MN_DrawMenu(CP_iteminfo *item_i, CP_itemtype *items)`
- **Purpose:** Draw all items for a menu
- **Inputs:** Menu layout and item definitions

### MN_GetActive / MN_MakeActive
- **Purpose:** Query or set which menu items are active (enabled/disabled)
- **Inputs/Outputs:** Menu info, item array, item index; outputs active count to `*nums`

### MN_DisplayInfo
- **Purpose:** Display descriptive text or help for the selected menu item
- **Inputs:** Item index (`which`)

### Menu-Specific Handlers
Functions like `MN_SerialMenu`, `MN_ModemMenu`, `MN_MusicMenu`, `MN_FXMenu`, `MN_StuffMenu` encapsulate sub-menu flows with their draw counterparts (`MN_DrawSerialMenu`, etc.).

### MN_EditListMenu / MN_HandleEntryList
- **Purpose:** Manage phonebook or configuration list (add, edit, delete entries)
- **Inputs:** Entry index for draw/print; list index for handler

### MN_DrawTBox
- **Signature:** `void MN_DrawTBox(int x, int y, int w, int h, boolean up)`
- **Purpose:** Draw a beveled text box (UI widget)

### MN_PlayMenuSnd
- **Signature:** `void MN_PlayMenuSnd(int which)`
- **Purpose:** Play a sound effect for menu actions (select, cancel, etc.)

### Utility Functions
- `MN_WaitKeyUp`: Block until user releases held key (debounce)
- `MN_DrawHalfStep`: Helper for smooth cursor animation
- `MN_DrawVolume`: Render volume slider for music/FX
- `MN_MusicVolume`, `MN_FXVolume`: Handle volume adjustment menus

## Control Flow Notes
This file is the configuration backbone for the menu system. It declares all menu hierarchies and item callbacks. The actual implementation (`st_menu.c`) uses these structures to:
1. **Init**: Load menu configurations (likely in setup phase)
2. **Frame**: Render active menu and process input on each tick
3. **Shutdown**: Free allocated menu resources

Menu callbacks (e.g., `MN_SerialMenu`, `MN_ManualEntry`) trigger sub-menus or actions. The system is event-driven: user input → `MN_HandleMenu` → callback invocation → next menu state.

## External Dependencies
- **`#include "rt_menu.h"`**: Provides base types:
  - `CP_iteminfo`: Menu layout metadata (x, y, item count, current position, indent level, font size)
  - `CP_itemtype`: Menu item (active flag, texture/string, hotkey letter, callback function pointer)
- **Inferred types** (defined elsewhere in rt_menu.h):
  - `CP_MenuNames` (char pointer array)
  - `menuptr` (function pointer typedef)
  - Globals: `colorname[]`, `NewGame`, `pickquick`, font pointers, window/buffer coords, game state flags

---

**Notes on Legacy Features:** The modem, serial port, and device selection menus reflect 1990s multiplayer infrastructure (dial-up networking, soundcard configuration). Modern builds likely disable or stub these.
