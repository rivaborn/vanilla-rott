## Key Functions / Methods

### ControlPanel
- Signature: `void ControlPanel(byte scancode)`
- Purpose: Entry point for in-game menu system; called when user presses ESC or F-keys during gameplay.
- Inputs: `scancode` — key that triggered menu (ESC or F1-F10)
- Outputs/Return: None; modifies game state and `playstate` based on user selections
- Side effects: Pauses game, saves/clears screen buffer, modifies `ingame`/`inmenu` flags, may change `playstate` to load game, end game, or return to demo
- Calls: `SetupMenuBuf()`, `SetUpControlPanel()`, `CP_F1Help()`, `CP_SaveGame()`, `CP_LoadGame()`, `CP_ControlMenu()`, `CP_EndGame()`, `CP_Quit()`, `CleanUpControlPanel()`, `SetupScreen()`
- Notes: Handles F1-F10 shortcuts for quick access to common functions; restores fizzlein effect on return to game.

### CP_MainMenu
- Signature: `menuitems CP_MainMenu(void)`
- Purpose: Displays and manages the main menu loop (new game, battle, restore, save, options, etc.).
- Inputs: None
- Outputs/Return: Returns which main menu item was selected (enum `menuitems`: newgame, battlemode, etc.)
- Side effects: Sets `StartGame` flag, modifies `ingame` flag, calls game initialization routines
- Calls: `SetupMenuBuf()`, `SetUpControlPanel()`, `DrawMainMenu()`, `HandleMenu()`, `DoMainMenu()`, `CP_NewGame()`, `CP_Quit()`, `CleanUpControlPanel()`, `ShutdownMenuBuf()`
- Notes: Loop continues until a valid selection is made or quit; `DoMainMenu()` redraws on return from sub-menus.

### HandleMenu
- Signature: `int HandleMenu(CP_iteminfo *item_i, CP_itemtype *items, void (*routine)(int w))`
- Purpose: Core menu navigation loop; handles user input (keyboard letters, arrow keys, mouse, joystick), item selection, and callbacks.
- Inputs: `item_i` — menu layout/metadata; `items` — array of menu items; `routine` — optional callback to redraw when cursor moves
- Outputs/Return: Returns selected item index, -1 for escape, or special values (PAGEUP, PAGEDOWN) for multi-page menus
- Side effects: Draws cursor, plays menu sounds (SD_MOVECURSORSND, SD_SELECTSND, SD_ESCPRESSEDSND), animates cursor frame cycling, modifies item active states
- Calls: `ReadAnyControl()`, `getASCII()`, `IN_ClearKeysDown()`, `WaitKeyUp()`, `RefreshMenuBuf()`, `DrawMenuBufItem()`, `MN_PlayMenuSnd()`, `ShowCursor()`, `HideCursor()`, `GetNumActive()`
- Notes: Supports half-step animation when moving between items; handles Home/End keys; supports letter shortcuts; blocks until user confirms or cancels; manages cursor animation state via `CursorNum`.

### ReadAnyControl
- Signature: `void ReadAnyControl(ControlInfo *ci)`
- Purpose: Unified input handler that polls keyboard, mouse, and joystick and populates a single `ControlInfo` structure.
- Inputs: Pointer to `ControlInfo` to fill
- Outputs/Return: Modifies `ci` with direction and button state
- Side effects: None (read-only input)
- Calls: `IN_UpdateKeyboard()`, `IN_ReadControl()`, `int386()` (mouse interrupt), `INL_GetJoyDelta()`, `IN_GetMouseButtons()`, `IN_JoyButtons()`
- Notes: Mouse and joystick movement is converted to cardinal directions (dir_North, dir_South, etc.); mouse recenters after reading; joystick overrides keyboard if active.

### CP_DisplayMsg
- Signature: `boolean CP_DisplayMsg(char *s, int number)`
- Purpose: Displays a message box with optional YES/NO buttons; handles user response.
- Inputs: `s` — message text (supports `\n` for line breaks); `number` — message type (determines sound and icon)
- Outputs/Return: Returns true if YES selected, false if NO/ESC
- Side effects: Plays sound based on message type (SD_WARNINGBOXSND, SD_QUESTIONBOXSND, etc.); sets `CP_Acknowledge` to CP_YES, CP_NO, or CP_ESC
- Calls: `RefreshMenuBuf()`, `ReadAnyControl()`, `MN_PlayMenuSnd()`, `DrawMenuBufPic()`, `DrawSTMenuBuf()`, `DrawMenuBufIString()`, `IN_ClearKeysDown()`, `IN_IgnoreMouseButtons()`
- Notes: Message type determines background pic and sound; number 13 is special (info only, no YES/NO); cursor controls direction selection.

### CP_NewGame
- Signature: `void CP_NewGame(void)`
- Purpose: New game initialization; prompts for difficulty selection and player character.
- Inputs: None
- Outputs/Return: None; sets `gamestate.battlemode`, `DefaultDifficulty`, and starts new game via `StartGame = true`
- Side effects: Ends any current game; displays difficulty/character selection menus; sets `playstate = ex_resetgame`
- Calls: `CP_DisplayMsg()`, `CP_PlayerSelection()`, `DrawNewGame()`, `HandleMenu()`, `DrawNewGameDiff()`, `EndGameStuff()`
- Notes: Uses randomized tough menus in full version (shareware has fixed menu); shows game in progress warning.

### CP_LoadGame / CP_SaveGame
- Signature: `int CP_LoadGame(int quick, int dieload)` / `int CP_SaveGame(void)`
- Purpose: Load or save game data with UI for slot selection and game naming.
- Inputs: `quick` — 1 for quick load (bypass dialog); `dieload` — loading due to death; none for save
- Outputs/Return: Returns 1 if successful load/save, 0 otherwise
- Side effects: Loads/saves game file; updates `SaveGamesAvail[]`, `SaveGameNames[][]`; may set `loadedgame = true`; modifies `MenuNum` for UI layout
- Calls: `DrawLoadSaveScreen()`, `HandleMenu()`, `GetSavedHeader()`, `LoadTheGame()`, `SaveTheGame()`, `US_LineInput()`, various draw functions
- Notes: Load checks for version compatibility; offers to delete incompatible saves; save allows user to name the game.

### CP_Control / CP_ControlMenu
- Signature: `void CP_Control(void)` / `void CP_ControlMenu(void)`
- Purpose: Device and input configuration menu; toggles mouse/joystick/gamepad, adjusts sensitivity, or enters custom mapping.
- Inputs: None
- Outputs/Return: None; modifies global control flags (`mouseenabled`, `joystickenabled`, etc.)
- Side effects: May call `CalibrateJoystick()`, enables/disables input devices; may trigger sub-menus for sensitivity or custom buttons
- Calls: `DrawCtlScreen()`, `HandleMenu()`, `DrawCtlButtons()`, `DoThreshold()`, `MouseSensitivity()`, `CP_Custom()`, `DrawControlMenu()`
- Notes: Tracks current active item in `CSTactive`; disables unavailable devices based on detection.

### CP_BattleModes
- Signature: `void CP_BattleModes(void)`
- Purpose: Battle mode selection menu; lists all 9 battle modes with descriptions and handles mode restrictions.
- Inputs: None
- Outputs/Return: None; sets `gamestate.battlemode`
- Side effects: Displays mode list with filtering (shareware/number of players); launches `CP_BattleMenu()` or `BattleGamePlayerSetup()` on selection
- Calls: `DrawBattleModes()`, `HandleMenu()`, `DrawBattleModeDescription()`, `CP_BattleMenu()`, `BattleGamePlayerSetup()`, `EndGameStuff()`
- Notes: Disables certain modes for shareware or single-player; sets `BATTLEMODE = true`.

### CP_SoundSetup
- Signature: `void CP_SoundSetup(void)`
- Purpose: Sound device configuration menu; allows selection and setup of sound/music cards with hardware parameters.
- Inputs: None
- Outputs/Return: None; configures sound system and saves settings
- Side effects: Initializes sound cards, may fade music, calls `SD_Startup()`, `MU_Startup()`
- Calls: `SetupMenuBuf()`, `DrawSoundSetupMainMenu()`, `HandleMenu()`, `SS_SoundMenu()`, `SS_MusicMenu()`, `FXVolume()`, `MusicVolume()`, `SS_Quit()`
- Notes: Extensive sub-menus for card type, port, DMA, IRQ selection; includes joystick calibration.

---

## Control Flow Notes
The menu system operates in discrete screens that block until user input:

1. **Initialization**: `SetUpControlPanel()` saves screen buffer, `SetupMenuBuf()` prepares display
2. **Menu Loop**: `DrawMenu()` renders items, `HandleMenu()` polls input and animates cursor, callbacks fire on selection
3. **Sub-menus**: Nested menu screens (e.g., Options → Detail → High/Medium/Low) use same `HandleMenu()` pattern
4. **State Transitions**: Main menu selections trigger state changes (`playstate = ex_resetgame` for new game, `ex_loadedgame` for load)
5. **Cleanup**: `CleanUpControlPanel()` restores screen and game state
6. **No Frame Loop**: All menu operations are synchronous; no tick-based rendering

The file does not participate in the main game loop's init/frame/update/render sequence; instead, it is a blocking modal overlay triggered by key press.

## External Dependencies
- **Input**: `rt_in.h` — `IN_UpdateKeyboard()`, `IN_ReadControl()`, `IN_GetMouseButtons()`, `INL_GetJoyDelta()`, `IN_JoyButtons()`, `CalibrateJoystick()`, keyboard scan code arrays
- **Resources**: `w_wad.h` — `W_GetNumForName()`, `W_CacheLumpNum()`, `W_CacheLumpName()`, `W_GetNameForNum()`
- **Graphics**: `rt_draw.h`, `rt_view.h`, `rt_vid.h` — `VWB_DrawPic()`, `VW_UpdateScreen()`, `VL_DrawPostPic()`, `DrawNormalSprite()`, `VL_FadeOut()`, `VL_FillPalette()`, screen coordinate macros
- **Sound**: `rt_sound.h` — `SD_Play()`, `SD_Startup()`, `SD_Shutdown()`, `MU_Startup()`, `MU_Shutdown()`, `MU_FadeOut()`, sound enumeration constants
- **Game State**: `rt_main.h` — `gamestate`, `playstate`, `locplayerstate`, `consoleplayer`, `numplayers`, `modemgame`
- **Game Logic**: `rt_game.h` — `BATTLE_SetOptions()`, `BATTLE_GetOptions()`, `GamePaused`, `RefreshPause`
- **Config**: `rt_cfg.h` — `WriteConfig()`, `GetPathFromEnvironment()`, `GetSavedMessage()`, `GetSavedHeader()`, `LoadTheGame()`, `SaveTheGame()`, various global settings
- **Utilities**: `z_zone.h` (memory), `rt_util.h`, `rt_str.h`, `rt_scale.h`, `rt_com.h`, `lumpy.h` (graphics headers), `modexlib.h` (VGA mode X)
- **Sound Hardware**: `fx_man.h` — sound device setup and initialization; DOS/system headers for hardware I/O and file operations
