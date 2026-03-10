# rott/rt_main.c

## File Purpose

This is the main entry point and core game loop orchestrator for Rise of the Triad. It initializes the engine (memory, input, sound, video), implements the primary state machine, and manages gameplay, menu, and cinematic transitions. Also handles command-line parsing, debugging, and auxiliary features like demo recording and screen capture.

## Core Responsibilities

- **Game initialization**: Set up memory manager, input system, sound/music, video mode, WAD files, palette/font cache
- **State machine**: Manage transitions between titles, gameplay, death, level completion, demos, and cinematics
- **Main game loop**: Orchestrate frame updates, rendering, and input polling during active play
- **Pause handling**: Implement pause state with optional screensaver
- **Input/keyboard**: Poll player input, handle debug keys, remote ridicule commands, and volume adjustments
- **Command-line parsing**: Interpret launch parameters (warp, turbo, sound setup, time limits, etc.)
- **Configuration**: Read/write game config, manage display settings, difficulty, and player state
- **Cinematics & demos**: Load cinematics, record/play demos, handle transitions
- **Screen capture**: Save gameplay screenshots in LBM or PCX format

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `exit_t` | enum | Game state codes (ex_titles, ex_stillplaying, ex_died, ex_battledone, etc.) |
| `objtype` | struct | Actor/dynamic object (position, state, flags, links) |
| `statobj_t` | struct | Static sprite object (lamps, items, gibs, switches) |
| `lbm_t` | struct | LBM image format (width, height, palette, pixel data) |
| `patch_t` | struct | Sprite patch format (bounding box, column offsets) |
| `font_t` | struct | Font definition (height, char widths, offsets) |
| `bmhd_t` | struct | IFF BMHD header (image width, height, planes) |
| `PCX_HEADER` | struct | PCX image file header |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `oldtime`, `gametime` | volatile int | global | Frame timing counters |
| `tedlevel`, `tedlevelnum`, `tedx`, `tedy` | bool/int | global | TED editor level launch parameters |
| `warp`, `warpx`, `warpy`, `warpa` | bool/int | global | Warp-to-level command-line parameters |
| `SCREENSHOTS`, `MONOPRESENT`, `MAPSTATS`, etc. | bool | global | Feature flags (debug, mono display, stats) |
| `dopefish` | bool | global | Easter egg trigger |
| `NoSound`, `IS8250` | int | global | Audio hardware flags |
| `CWD` | char[40] | global | Current working directory string |
| `timelimit`, `maxtimelimit`, `timelimitenabled` | int | global | Time-limit mode parameters |
| `playstate` | exit_t | global | Current game state (managed by GameLoop) |
| `savename` | char[13] | static | Screenshot filename buffer (conditional compile) |
| `turbo`, `NoWait`, `startlevel`, `demonumber` | static int | static | Game launch modifiers |
| `quitactive` | static bool | static | Quit confirmation state |

## Key Functions / Methods

### main
- **Signature:** `void main(void)`
- **Purpose:** Application entry point; initialize engine subsystems and launch main game loop
- **Inputs:** Command-line arguments (_argc, _argv)
- **Outputs/Return:** None (calls exit at termination)
- **Side effects:** Initializes global state, allocates memory, starts music/sound, enters graphics mode, calls GameLoop() and QuitGame()
- **Calls:** DrawRottTitle, CheckCommandLineParameters, Z_Init, IN_Startup, ReadConfig, ReadSETUPFiles, SetupWads, BuildTables, GetMenuInfo, MU_Startup, SD_SetupFXCard, SD_Startup, Init_Tables, InitializeRNG, InitializeMessages, LoadColorMap, I_StartupTimer, I_StartupKeyboard, VL_SetVGAPlaneMode, VL_SetPalette, SetViewSize, CP_SoundSetup, BATTLE_SetOptions, PlayTurboGame, ApogeeTitle, DopefishTitle, GameLoop, QuitGame
- **Notes:** Conditional compilation for shareware vs. registered versions; handles sound setup failures gracefully

### GameLoop
- **Signature:** `void GameLoop(void)`
- **Purpose:** Main game state machine; orchestrate transitions between titles, gameplay, death, demos, and cinematics
- **Inputs:** None (reads global playstate, consoleplayer, numplayers)
- **Outputs/Return:** None (runs until playstate changes to exit condition)
- **Side effects:** Modifies playstate, initializes/shuts down battle mode, loads/frees level data, manages music state
- **Calls:** BATTLE_Shutdown, MU_StartSong, VL_FadeIn, VL_FillPalette, BattleLevelCompleted, Z_FreeTags, AdjustMenuStruct, CalcTics, InitCharacter, InitializeMessages, BATTLE_GetSpecials, BATTLE_SetOptions, SetupGameMaster, SetupGamePlayer, BATTLE_Init, PlayCinematic, SetupGameLevel, SetupScreen, PlayLoop, Died, CheckForQuickLoad, CheckHighScore, LevelCompleted, GetNextMap, PlayMovie, CP_MainMenu, etc.
- **Notes:** Massive switch statement on playstate; handles both single-player and network game modes; includes version checks for network compatibility

### PlayLoop
- **Signature:** `void PlayLoop(void)`
- **Purpose:** Execute active gameplay frame loop; poll input, update objects, render, and handle menu/pause
- **Inputs:** None (reads global state: playstate, GamePaused, controlupdatestarted, demoplayback)
- **Outputs/Return:** None (runs while playstate == ex_stillplaying)
- **Side effects:** Updates player position, game objects, camera, display; modifies playstate on menu/quit
- **Calls:** DrawPlayScreen, DoLoadGameSequence, StartupClientControls, ShutdownClientControls, UpdateGameObjects, PauseLoop, ThreeDRefresh, UpdateScreenSaver, UpdateClientControls, PollKeyboard, AnimateWalls, DoSprites, DoAnimatedMaskedWalls, UpdatePlayers, DrawTime, ControlPanel, Z_CheckHeap, AdaptDetail
- **Notes:** Main inner loop that runs at ~70 Hz; handles pause, menu entry (Escape), demo playback exit, and detail level adaptation

### UpdateGameObjects
- **Signature:** `void UpdateGameObjects(void)`
- **Purpose:** Advance game time, move actors, process triggers, evaluate battle conditions
- **Inputs:** None (reads fasttics, demoplayback, numclocks, TRIGGER array)
- **Outputs/Return:** None (modifies actor positions, gamestate.TimeCount)
- **Side effects:** Moves doors/elevators, updates lightning, processes actor AI via DoActor, evaluates battle status, handles time limits
- **Calls:** UpdateClientControls, PollControls, CalcTics, MoveDoors, ProcessElevators, MovePWalls, UpdateLightning, TriggerStuff, CheckCriticalStatics, DoActor, BATTLE_CheckGameStatus, CheckForSyncCheck
- **Notes:** Runs at logical frame rate (not rendering rate); loop runs oldpolltime < oldtime for frame-rate independent gameplay

### PauseLoop
- **Signature:** `void PauseLoop(void)`
- **Purpose:** Update logic while game is paused; optionally run screensaver
- **Inputs:** None (reads GamePaused, RefreshPause, ticcount, pausedstartedticcount, blanktime)
- **Outputs/Return:** None
- **Side effects:** Processes pause unpause, updates screensaver, modifies RefreshPause flag
- **Calls:** StopWind, UpdateClientControls, CheckUnPause, CheckForSyncCheck, CalcTics, PollControls, StartupScreenSaver, UpdateScreenSaver
- **Notes:** Alternative to normal gameplay update when paused; respects sync checks for network

### PollKeyboard
- **Signature:** `void PollKeyboard(void)`
- **Purpose:** Handle all keyboard input during gameplay: debug keys, menu shortcuts, volume/detail adjustment, messaging
- **Inputs:** None (reads Keyboard array, LastScan)
- **Outputs/Return:** None (modifies global state: gamestate.autorun, DetailLevel, MessagesEnabled, gamma, volumes)
- **Side effects:** Toggles autorun, adjusts detail/messages/gamma/volumes, triggers map/save dialogs, enables message mode, saves screenshots
- **Calls:** IN_UpdateKeyboard, CheckDebug, DebugKeys, AddMessage, DeletePriorityMessage, StopWind, DoMap, SetupScreen, DoBossKey, SaveScreen, IN_WaitForKey, IN_UpdateKeyboard
- **Notes:** Extensive conditional key handling (F5–F12, bracket keys for volume); supports both single-player and battle-mode contexts

### CheckCommandLineParameters
- **Signature:** `void CheckCommandLineParameters(void)`
- **Purpose:** Parse command-line arguments and initialize corresponding flags/values
- **Inputs:** _argc, _argv
- **Outputs/Return:** None (modifies global flags and parameters)
- **Side effects:** Sets tedlevel, NoWait, NoSound, turbo, warp, dopefish, MAPSTATS, IS8250, timelimit, networkgame, numplayers, etc.
- **Calls:** CheckParm, US_CheckParm, ParseNum, TextMode, printf, InitROTTNET, exit
- **Notes:** Prints help if "?" or "HELP" passed; calls InitROTTNET for multiplayer setup; defines multiple version strings for shareware/registered/deluxe

### InitCharacter
- **Signature:** `void InitCharacter(void)`
- **Purpose:** Reset player state for new game: health, lives, score, weapon, triads, mode flags
- **Inputs:** None (reads locplayerstate, gamestate.battlemode, timelimitenabled, startlevel)
- **Outputs/Return:** None
- **Side effects:** Resets player health, lives, score, weapon, difficulty, and frame counters
- **Calls:** MaxHitpointsForCharacter, ClearTriads, UpdateScore
- **Notes:** Called on new game start and game reset

### QuitGame
- **Signature:** `void QuitGame(void)`
- **Purpose:** Fade out music, print debug/end-screen info, shut down all subsystems, and exit
- **Inputs:** None
- **Outputs/Return:** None (calls exit(0))
- **Side effects:** Fades music, writes config, shuts down keyboard/timer/sound/memory, displays end screen
- **Calls:** MU_FadeOut, MU_FadeActive, PrintMapStats, PrintTileStats, TextMode, ShutDown, exit
- **Notes:** Conditional debug output (DEVELOPMENT flag); displays registered vs. shareware end screen

### PlayCinematic
- **Signature:** `void PlayCinematic(void)`
- **Purpose:** Load and display intro/interstitial cinematics for specific level milestones
- **Inputs:** None (reads gamestate.mapon, tedlevel, turbo)
- **Outputs/Return:** None
- **Side effects:** Displays LBM graphics, plays music and SFX, waits for input
- **Calls:** MU_StartSong, VL_FadeOut, VL_ClearBuffer, DrawNormalSprite, FlipPage, VL_FadeIn, I_Delay, DoInBetweenCinematic, IN_UpdateKeyboard, SD_Play
- **Notes:** Skipped if tedlevel or turbo active; different cinematics for episode starts (levels 0, 8, 16, 24)

### PutBytes
- **Signature:** `int PutBytes(unsigned char *ptr, unsigned int bytes)`
- **Purpose:** RLE-compress a scan line for PCX output
- **Inputs:** ptr (scan-line data), bytes (number of bytes to encode)
- **Outputs/Return:** 0 on success
- **Side effects:** Writes to global bptr and totalbytes (PCX buffer state)
- **Calls:** None
- **Notes:** Encodes repeated bytes as (0xc0 | count, value); used by WritePCX

## Control Flow Notes

**Initialization & Startup (main):**
- Parse command line → Load config/WAD files → Initialize memory/input/sound/video → Set initial game state (ex_titles) → Call GameLoop

**GameLoop State Machine:**
- **ex_titles:** Display intro movies, credits, demos; transition to menu or gameplay
- **ex_resetgame:** Initialize character, battle mode, level; transition to ex_stillplaying
- **ex_stillplaying:** Call PlayLoop (active gameplay)
- **ex_died / ex_completed / ex_skiplevel / ex_secretlevel / ex_bossdied:** End level, show cutscene/score, transition to next level or titles
- **ex_warped:** Teleport to specified level; transition to ex_stillplaying
- **ex_demorecord / ex_demoplayback:** Record or play demo; transition to ex_stillplaying or ex_demodone
- **ex_battledone / ex_gameover:** End game, show high-score, transition to ex_titles

**PlayLoop (Gameplay Frame Loop):**
- Poll keyboard → Update game objects (actors, doors, triggers) → Check pause state → Render 3D view → Update display

**Shutdown (QuitGame):**
- Fade music → Print debug info → ShutDown() → exit(0)

## External Dependencies

### Notable Includes / Imports
- **Engine core:** rt_def.h (constants, types), rt_actor.h (actor management), rt_stat.h (static objects), rt_vid.h (video/palette), rt_game.h (game state)
- **Rendering:** rt_draw.h, rt_view.h, rt_scale.h (3D rendering), modexlib.h (VGA mode X), rt_dr_a.h
- **Input:** rt_in.h (input polling), isr.h (interrupt handlers)
- **Audio:** music.h, fx_man.h, rt_sound.h (sound/music control)
- **Game systems:** rt_door.h (doors), rt_floor.h (elevators), rt_playr.h (player state), rt_menu.h (menus), rt_map.h (map/level), rt_ted.h (TED editor), rt_net.h, rottnet.h (networking), cin_main.h (cinematics)
- **Utilities:** z_zone.h (memory manager), w_wad.h (WAD loader), rt_util.h, rt_msg.h (messages), rt_cfg.h (config), rt_rand.h (RNG), rt_debug.h (debug macros)
- **Platform:** dos.h, conio.h, graph.h, direct.h, bios.h, process.h, fcntl.h, io.h, malloc.h (DOS/Watcom SDK)
- **Data:** scriplib.h (script parsing), develop.h, version.h, memcheck.h (memory debugging)

### Symbols Defined Elsewhere
- **Game state:** gamestate (global), locplayerstate, PLAYERSTATE, player, consoleplayer, numplayers
- **Input:** LastScan, Keyboard[], buttonpoll[], buttonscan[]
- **Rendering:** bufferofs, displayofs, origpal, ylookup[]
- **Audio:** MUvolume, FXvolume, MusicMode, FXMode
- **Battle:** BATTLE_Options, BATTLE_ShowKillCount, BATTLE_Init, BATTLE_CheckGameStatus, BATTLEMODE
- **Demo/Demo state:** demorecord, demoplayback, demo-loading functions
- **Level data:** mapplanes, tilemap, TRIGGER[], Clocks[], firstactive, firstemptystat, etc.
