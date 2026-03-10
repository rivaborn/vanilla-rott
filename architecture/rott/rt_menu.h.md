# rott/rt_menu.h

## File Purpose
Public header for the menu and control panel system in Rise of the Triad. Declares all menu structures, UI rendering functions, game state navigation (main menu, options, load/save), and multiplayer setup handlers.

## Core Responsibilities
- Menu rendering and item management (fonts, positioning, active states, texture display)
- Game state transitions (main menu → new game, load, settings, battle modes, quit)
- Player selection, name entry, and multiplayer configuration (teams, CTF, modems)
- Save/load game functionality including quick-save/undo
- Sound and audio options configuration
- Color selection and display options (detail level, bobbing, flip speed)
- Input capture and menu navigation (keyboard scanning, hotkeys, multi-page menus)
- Screen resource allocation/deallocation for menu rendering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `mn_fontsize` | enum | Font size variants for menu text rendering (tiny, 8x8, small, large) |
| `CP_iteminfo` | struct | Menu item layout: position (x,y), count, current position, font size, names array |
| `CP_itemtype` | struct | Menu item properties: active state, texture, hotkey letter, callback routine pointer |
| `menuitems` | enum | Menu sections: newgame, battlemode, loadgame, savegame, control, orderinfo, viewscores, backtodemo, quit |
| `CP_MenuNames` | typedef | Pointer type for menu item name strings |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `NewGame` | boolean | global | Flag for starting new game |
| `MainMenu[]` | CP_itemtype | global | Main menu item definitions |
| `pickquick` | boolean | global | Quick selection flag |
| `CurrentFont`, `tinyfont`, `newfont1`, `smallfont`, `bigfont` | font_t* | global | Font pointers for menu text rendering |
| `IFont` | cfont_t* | global | Colored font for menus |
| `PrintX`, `PrintY` | int | global | Text cursor position |
| `WindowX`, `WindowY`, `WindowH`, `WindowW` | int | global | Menu window bounds |
| `px`, `py`, `bufferwidth`, `bufferheight` | int | global | Screen/buffer dimensions |
| `loadedgame` | boolean | global | Flag: game currently loaded from save |
| `FXMode`, `MusicMode` | int | global | Audio settings |
| `AutoDetailOn`, `DoubleClickOn`, `BobbinOn` | boolean | global | Display options |
| `DetailLevel`, `Menuflipspeed` | int | global | Visual settings |
| `ingame`, `inmenu` | boolean | global | Game state flags |
| `scancode` | int | global | Current keyboard input |
| `quicksaveslot` | int | global | Quick-save slot index |
| `colorname[]` | char* | global | Color name strings (defined elsewhere) |

## Key Functions / Methods

### SetUpControlPanel / CleanUpControlPanel
- Signature: `void SetUpControlPanel(void)` / `void CleanUpControlPanel(void)`
- Purpose: Lifecycle setup and teardown for menu system
- Side effects: Allocates/deallocates menu resources; manages screen state

### ControlPanel / CP_MainMenu / DoMainMenu
- Signature: `void ControlPanel(byte scancode)` / `menuitems CP_MainMenu(void)` / `void DoMainMenu(void)`
- Purpose: Main menu control flow; interpret user input and dispatch to menu handlers
- Inputs: scancode (keyboard input)
- Outputs: menuitems enum indicating selected menu section
- Side effects: Updates global game state, may trigger load/save, mode changes

### DrawMenu / DrawMainMenu
- Signature: `void DrawMenu(CP_iteminfo *item_i, CP_itemtype *items)` / `void DrawMainMenu(void)`
- Purpose: Render menu items to screen with current font and colors
- Inputs: item_i (layout/positioning), items (item definitions with names/textures)
- Side effects: Screen update/rendering

### AllocateSavedScreenPtr / FreeSavedScreenPtr
- Signature: `void AllocateSavedScreenPtr(void)` / `void FreeSavedScreenPtr(void)`
- Purpose: Manage off-screen buffer for menu background preservation
- Side effects: Heap allocation/deallocation

### CP_LoadGame / QuickSaveGame / UndoQuickSaveGame
- Signature: `int CP_LoadGame(int quick, int dieload)` / `void QuickSaveGame(void)` / `void UndoQuickSaveGame(void)`
- Purpose: Load game state from disk or manage quick-save slots
- Inputs: quick (1 for quick-load), dieload (load-on-death flag)
- Outputs: Status code (success/failure)
- Side effects: File I/O; updates game state; restores player position/inventory

### CP_ColorSelection / CP_SoundSetup
- Signature: `int CP_ColorSelection(void)` / `void CP_SoundSetup(void)`
- Purpose: Display color/palette or sound configuration menus
- Outputs: Selected color index (CP_ColorSelection)
- Side effects: Updates audio or visual settings

### BattleGamePlayerSetup / BattleNoTeams / BattleTeams / CP_BattleModes
- Signature: Various void or int returns
- Purpose: Configure multiplayer battle modes (teams, CTF, modem comms)
- Side effects: Updates multiplayer configuration; may show player selection dialogs

### CP_PlayerSelection / CP_EnterCodeNameMenu
- Signature: `int CP_PlayerSelection(void)` / `int CP_EnterCodeNameMenu(void)`
- Purpose: UI for selecting player character or entering player name
- Outputs: Selected player ID or name index
- Side effects: Updates player configuration

### HandleMultiPageCustomMenu
- Signature: `int HandleMultiPageCustomMenu(char **names, int amount, int curpos, char *title, void (*routine)(int w), void (*redrawfunc)(void), boolean exitonselect)`
- Purpose: Generic paginated menu handler with custom callback and redraw routine
- Inputs: Menu item names, item count, current position, title, callback routine, redraw function, exit-on-select flag
- Outputs: Selected item index
- Side effects: Calls callback routine on item selection; calls redraw function each frame

### ReadAnyControl / WaitKeyUp / CP_CheckQuick
- Signature: `void ReadAnyControl(ControlInfo *ci)` / `void WaitKeyUp(void)` / `boolean CP_CheckQuick(byte scancode)`
- Purpose: Input polling and quick-save hotkey detection
- Inputs: ControlInfo structure, scancode
- Outputs: Filled ControlInfo or quick-save flag
- Notes: ReadAnyControl abstracts keyboard/joystick/mouse input; CP_CheckQuick checks for quick-load hotkey

### Message / CP_DisplayMsg / DisplayInfo
- Signature: `void Message(char *string)` / `boolean CP_DisplayMsg(char *s, int number)` / `void DisplayInfo(int which)`
- Purpose: Show messages, dialogs, or info screens
- Inputs: Message string, info type ID
- Outputs: CP_DisplayMsg returns boolean (user dismissed?)
- Side effects: Screen drawing; may wait for user input

### Trivial Helpers
- `MN_PlayMenuSnd(int which)` – Play menu sound effect
- `GetMenuInfo()` / `WriteMenuInfo()` – Load/save menu configuration
- `GetNumActive()` / `MN_GetActive()` / `MN_MakeActive()` – Menu item active-state queries
- `AdjustMenuStruct()` / `MenuFixup()` – Menu structure validation/adjustment
- `GetEpisode(int level)` – Derive episode from level ID
- `getASCII()` – Read ASCII input from keyboard queue
- `CP_ViewScores()` – Display high scores
- `CP_CaptureTheTriadError()` / `CP_TeamPlayErrorMessage()` – Multiplayer error dialogs

## Control Flow Notes
This is a UI subsystem without single linear flow. **Initialization** happens in `SetUpControlPanel()` and `DrawMainMenu()`. **Per-frame** input is handled by `ControlPanel()` or menu-specific handlers (e.g., `CP_MainMenu()`, `CP_LoadGame()`) which interpret `scancode` and call item callback routines. **Rendering** happens via `DrawMenu()` or menu-specific draw functions. **Shutdown** occurs in `CleanUpControlPanel()` before game loop exit or level transitions. The `HandleMultiPageCustomMenu()` function serves as a reusable dispatcher for complex multi-page dialogs.

## External Dependencies
- **lumpy.h** — pic_t, lpic_t, font_t, cfont_t, patch_t, transpatch_t (graphics structures)
- **rt_in.h** — ControlInfo, KeyboardDef, JoystickDef, Motion, Direction, ControlType enums; input query functions (IN_ReadControl, IN_WaitForKey, etc.)
- **Undefined here** — boolean, byte, word (primitive types defined elsewhere, likely develop.h or standard headers); file I/O functions (defined elsewhere); rendering functions; game state globals
