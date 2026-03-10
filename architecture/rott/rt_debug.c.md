# rott/rt_debug.c

## File Purpose

Implements cheat code detection and execution for the Rise of the Triad game engine. Processes keyboard input to recognize cheat sequences and triggers corresponding debug/cheat effects (godmode, weapons, level warping, demo recording, etc.). Also provides frame-by-frame development debug keys when DEVELOPMENT mode is enabled.

## Core Responsibilities

- Detect and validate cheat code sequences from keyboard input
- Execute cheat effects (god mode, invulnerability, weapons, armor, powerups)
- Handle level warping and game state modifications
- Support demo recording, playback, and jukebox access
- Manage game state toggles (light diminishing, fog, HUD, missile cam, floor/ceiling rendering)
- Provide frame-by-frame debug controls for development builds
- Spawn bonus items and manage player health/armor/keys via cheat spawning

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `CodeStruct` | struct | Represents a single cheat code: key sequence string and its length |
| Cheat code enum (ENABLECHEAT, GODMODEPWUP, etc.) | enum | Enum values identifying each cheat code type; supports dual key variants (ALT versions) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Codes[]` | `CodeStruct[MAXCODES + 4]` | global/static | Master lookup table of all cheat codes with their key sequences and lengths |
| `DebugOk` | (external, likely int/bool) | global | Flag controlling whether cheats are enabled; only toggle-cheat codes work when disabled |

## Key Functions / Methods

### CheckDebug
- Signature: `void CheckDebug(void)`
- Purpose: Main entry point called once per frame to check if any cheat codes were entered; dispatches to code checks based on current game state
- Inputs: None (reads global keyboard state)
- Outputs/Return: None
- Side effects: Calls `CheckCode()` for relevant cheat indices based on `DebugOk` flag and current playback/recording state
- Calls: `CheckCode()` (multiple times with different indices)
- Notes: If cheats disabled, only checks codes 0 and 1 (ENABLECHEAT variants). If demo is playing, no checks. If demo is recording, only DEMORECORD and DEMOEND are checked.

### CheckCode
- Signature: `void CheckCode(int which)`
- Purpose: Determines if a specific cheat code was entered by examining the keyboard letter queue backwards; if matched, executes the corresponding cheat handler
- Inputs: `which` – index into `Codes[]` array
- Outputs/Return: None
- Side effects: Modifies global state via cheat handlers (player flags, health, weapons, level, etc.); calls `AddMessage()` to display feedback
- Calls: Large switch statement dispatching to handler functions (`DoGodMode()`, `DoWarp()`, `DoItemCheat()`, etc.)
- Notes: Uses circular buffer `LetterQueue[]` with wrapping via bitwise AND with `(MAXLETTERS-1)`. Compares uppercase versions of characters. Clears matched code from queue to prevent re-triggering.

### DebugKeys
- Signature: `int DebugKeys(void)` (returns 1 if a debug key was handled, 0 otherwise)
- Purpose: Handles single-key debug controls during development (only when DEVELOPMENT == 1); provides frame-by-frame testing shortcuts
- Inputs: None (reads `Keyboard[]` array)
- Outputs/Return: 1 if a key was processed, 0 otherwise
- Side effects: Modifies player state, triggers level end, records FPS, spawns items, toggles god/armor modes, etc.; clears keyboard state after processing
- Calls: `DoGodMode()`, `DoWarp()`, `HurtPlayer()`, `EndLevel()`, `IN_UpdateKeyboard()`, `IN_ClearKeysDown()`, etc.
- Notes: Only active in development builds. Includes FPS counter (F key), god mode (G), warp (W), hurt self (H), end level (Z), cycle powerups (P), cycle armor (A), outfit player (O), kill self (K), item cheat (I), demo commands (R/E/D).

### DoWarp
- Signature: `void DoWarp(void)`
- Purpose: Provides interactive level selection UI allowing the player to warp to any level
- Inputs: None (reads menu selection input)
- Outputs/Return: None
- Side effects: Sets `gamestate.mapon`, `gamestate.episode`, `playstate = ex_warped`; shuts down/restarts client controls; fades screen; manages music position
- Calls: `MU_StoreSongPosition()`, `MU_StartSong()`, `StopWind()`, `SetupMenuBuf()`, `SetUpControlPanel()`, `CP_LevelSelectionMenu()`, `CleanUpControlPanel()`, etc.
- Notes: If user cancels or selects current level, returns to normal game state and restores music.

### DoJukeBox
- Signature: `void DoJukeBox(void)`
- Purpose: Provides music/sound selection menu during gameplay
- Inputs: None (reads menu input)
- Outputs/Return: None
- Side effects: Suspends gameplay, shows jukebox menu, restores screen and input state
- Calls: `StopWind()`, `ShutdownClientControls()`, `SetupMenuBuf()`, `SetUpControlPanel()`, `MU_JukeBoxMenu()`, etc.
- Notes: Simple wrapper around menu system; restores screen state on exit.

### CheatSpawnItem
- Signature: `void CheatSpawnItem(int item)`
- Purpose: Spawns a bonus/powerup item at the player's current location
- Inputs: `item` – stat type (stat_godmode, stat_bulletproof, etc.)
- Outputs/Return: None
- Side effects: Calls `SpawnStatic()`, modifies `LASTSTAT` (z position, flags)
- Calls: `SpawnStatic()`, `MakeStatActive()` (on LASTSTAT)
- Notes: Sets `FL_ABP` flag and z-coordinate to player's height.

### DoGodMode, DoItemCheat, DoSomeItemCheat, etc.
- Signature: Varied (most take `void`)
- Purpose: Individual cheat effect handlers
- Inputs: Varies (most none)
- Outputs/Return: None
- Side effects: Modify player flags, health, weapons, keys, game state, or call other handlers; send `MSG_CHEAT` messages
- Calls: Helpers like `AddMessage()`, `HealPlayer()`, `DrawKeys()`, `CheatSpawnItem()`, `GivePoints()`, `SpawnStatic()`, etc.
- Notes: Each corresponds to one or more cheat code entries. Many are straightforward state toggles or item spawns.

## Control Flow Notes

**Cheat Code Pipeline:**
1. Frame-by-frame, game calls `CheckDebug()` (called from game loop via external caller, likely in main game loop)
2. `CheckDebug()` dispatches to `CheckCode()` for relevant code indices
3. `CheckCode()` scans keyboard letter buffer for a match; if found, executes the handler
4. Handlers modify game state and send feedback messages

**Development Debug Keys (DEVELOPMENT == 1):**
- `DebugKeys()` is called separately from the cheat code path and provides single-key shortcuts

**Integration Point:**
- Not inferable from this file exactly where `CheckDebug()` is called, but likely in main game loop or input handler (rt_game.c, rt_main.c, or similar)

## External Dependencies

**Notable Includes / Imports:**
- `rt_def.h` – core engine definitions (flags, types, constants)
- `rt_game.h` – game state, player management, scoring, rendering  
- `rt_menu.h` – menu system (control panel, level selection, jukebox)
- `rt_in.h` – input management
- `rt_playr.h` – player object structure
- `rt_build.h` – level/build data
- `rt_draw.h`, `rt_vid.h` – rendering
- `rt_view.h` – viewport/camera
- `rt_sound.h`, `rt_msg.h`, `rt_net.h`, `rt_stat.h`, `rt_map.h` – sound, messaging, networking, entity/map state
- `isr.h` – keyboard/interrupt input (Keyboard array, IN_UpdateKeyboard, etc.)
- `z_zone.h` – memory management
- `develop.h` – development utilities
- `memcheck.h` – memory debugging

**Defined Elsewhere, Used Here:**
- Global arrays/vars: `Keyboard[]`, `LetterQueue[]`, `PLAYER[]`, `PLAYERSTATE[]`, `LASTSTAT`, `MAPSPOT()`
- Functions: `SpawnStatic()`, `MakeStatActive()`, `AddMessage()`, `DamageThing()`, `HealPlayer()`, `DrawKeys()`, `GivePoints()`, `CheatMap()`, `CP_LevelSelectionMenu()`, `MU_JukeBoxMenu()`, `MU_StartSong()`, `MU_RestoreSongPosition()`, `SaveDemo()`, `LoadDemo()`, `DemoExists()`, `RotationFun()`, `ThreeDRefresh()`, `DoSprites()`, `CalcTics()`, etc.
- Globals: `player`, `gamestate`, `locplayerstate`, `godmode`, `missilecam`, `HUD`, `fandc`, `demorecord`, `demoplayback`, `ludicrousgibs`, `DebugOk` (extern), `CurrentFont`, `smallfont`
