# rott/rt_main.h

## File Purpose
Main engine header defining the global game state (`gametype`), core initialization/update/shutdown functions, and central game variables. Acts as the primary interface for the game's main loop and state management.

## Core Responsibilities
- Define the central game state structure tracking score, kills, inventory, difficulty, and battle settings
- Declare lifecycle functions for game initialization, updates, and shutdown
- Export global debugging and development mode flags
- Manage special power-up timer state
- Track game version information (shareware vs. registered)
- Expose main loop control functions (pause, cinematic playback, screen saving)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `specials` | struct | Tracks duration and respawn times for 8 power-up types (god mode, dog mode, mushrooms, etc.) |
| `gametype` | struct | Central game state: game progress (kills/secrets/treasure counts), difficulty, violence level, team mode, battle options, special timers, player inventory |
| `version_type` | enum | Game variant (ROTT_SHAREWARE, ROTT_REGISTERED, ROTT_SUPERCD, ROTT_SITELICENSE) |
| `vl_*` (volume enum) | enum | Audio volume levels (low, medium, high, excessive) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gamestate` | gametype | global | Primary game state container |
| `doublestep`, `tedlevelnum`, `tedx`, `tedy` | int | global | Level editor / debug positioning |
| `fizzlein` | boolean | global | Screen effect flag |
| `NoSound`, `timelimit`, `timelimitenabled` | int/boolean | global | Audio and game time settings |
| `polltime`, `oldpolltime`, `oldtime` | int/volatile int | global | Timing for frame sync |
| `DebugOk`, `newlevel` | boolean | global | Debug mode and level transition flags |
| `SCREENSHOTS`, `MEMORYTEST`, `MODEMTEST`, etc. | boolean | global | Feature/test flags (9 total) |
| `CWD` | char[40] | global | Current working directory |

## Key Functions / Methods

### QuitGame
- **Signature:** `void QuitGame( void )`
- **Purpose:** Exit the game cleanly
- **Inputs:** None
- **Outputs/Return:** None (exits program)
- **Side effects:** Program termination

### InitCharacter
- **Signature:** `void InitCharacter( void )`
- **Purpose:** Initialize player character state and properties
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies game state

### UpdateGameObjects
- **Signature:** `void UpdateGameObjects( void )`
- **Purpose:** Update all dynamic game entities each frame
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies actor/sprite state, collision detection

### ShutDown
- **Signature:** `void ShutDown( void )`
- **Purpose:** Clean shutdown (graphics, sound, memory)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Releases resources, restores system state

### PauseLoop
- **Signature:** `void PauseLoop( void )`
- **Purpose:** Handle pause menu and game pause state
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Input polling, pause flag changes

### PlayCinematic
- **Signature:** `void PlayCinematic( void )`
- **Purpose:** Play in-game cinematics/animations
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Video playback, audio

### SetupWads
- **Signature:** `void SetupWads( void )`
- **Purpose:** Load and initialize WAD (game data) files
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** File I/O, memory allocation

**Notes:** `SaveScreen()` conditionally compiled (requires `SAVE_SCREEN` flag); trivial helpers not listed.

## Control Flow Notes
File represents engine initialization/main-loop interface: `InitCharacter()` and `SetupWads()` during startup; `UpdateGameObjects()` in main loop; `PauseLoop()` for pause state; `PlayCinematic()` for cinematics; `ShutDown()`/`QuitGame()` for cleanup. Global `gamestate` persists across frames. The `battle_type` structure (from rt_battl.h) indicates multiplayer battle mode integration.

## External Dependencies
- **develop.h** — Development flags (DEBUG, SHAREWARE, WEAPONCHEAT, etc.)
- **rt_def.h** — Engine constants (screen geometry, angles, map sizes, actor limits, global flags)
- **rottnet.h** — Networking primitives (MAXPLAYERS, rottcom_t structure)
- **rt_battl.h** — Battle system types (battle_type, battle_status)

**Defined elsewhere:** Function implementations, actor/sprite systems, rendering backend, input handling, audio subsystem.
