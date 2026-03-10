# rott/rt_main.c — Enhanced Analysis

## Architectural Role

rt_main.c is the **engine orchestrator and state supervisor** for Rise of the Triad. It coordinates the initialization and shutdown of a vast subsystem ecosystem (memory, input, audio, video, networking), implements the game's primary state machine (`exit_t` enum), and orchestrates the frame loop that drives gameplay, menus, and cinematics. This file is the "conductor" of the engine—it doesn't implement individual systems, but rather sequences their startup, manages transitions between game modes, and ensures proper cleanup on exit.

## Key Cross-References

### Incoming (Who Depends on This File)
- **Operating system entry point**: rt_main exports `main()`, the only external entry point for the DOS executable
- **Launchers / batch files**: Command-line parameters flow from OS/batch files into `CheckCommandLineParameters()`, affecting initialization branches
- **Configuration / persistence**: `ReadConfig()` and `ReadSETUPFiles()` wire saved settings back into global state (difficulty, sound settings, player name)
- **Version checking**: SHAREWARE, SUPERROTT, SITELICENSE, and REGISTERED compile-time flags control product branching (Apogee licensing model)

### Outgoing (What This File Calls Into)

**Initialization Sequence (main → subsystems):**
- **Memory:** `Z_Init()` (z_zone.h) — reserves heap before anything else allocates
- **Input:** `IN_Startup()` → `I_StartupKeyboard()` / `I_StartupTimer()` (rt_in.h, isr.h)
- **Video:** `VL_SetVGAPlaneMode()` / `VL_SetPalette()` (rt_vid.h) — graphics mode setup; calls `SetViewSize()` to configure viewport
- **Audio:** `MU_Startup()` (music.h) → `SD_SetupFXCard()` / `SD_Startup()` (rt_sound.h) — music and sound FX initialization with fallback/error handling
- **Game systems:** `BuildTables()` (rt_draw.h), `InitializeRNG()` (rt_rand.h), `InitializeMessages()` (rt_msg.h), `LoadColorMap()` (palette cache)
- **Networking:** `InitROTTNET()` (rottnet.h) — only called if command-line specifies multiplayer
- **Menus:** `GetMenuInfo()`, `ReadConfig()`, `ReadSETUPFiles()` (rt_menu.h, rt_cfg.h)

**Main Loop Orchestration (GameLoop → subsystems):**
- **State management:** `BATTLE_Init()` / `BATTLE_Shutdown()`, `BATTLE_SetOptions()` / `BATTLE_GetOptions()` (rt_battl.h) — battle-mode state machine
- **Level loading:** `SetupGameLevel()` (rt_game.h) — initializes level data, actors, triggers
- **Audio:** `MU_StartSong()` (music.h) — transitions music between levels
- **Cinematics:** `PlayCinematic()` (internal) — shows interstitial story cinematics; calls `PlayMovie()` (cin_main.h)
- **Core gameplay:** `PlayLoop()` (internal) — drives the active frame loop
- **High-score:** `CheckHighScore()` (rt_game.h) — end-of-game flow

**Frame Loop Details (PlayLoop → subsystems):**
- **Rendering:** `DrawPlayScreen()` (rt_draw.h) → `ThreeDRefresh()` (rt_view.h) — 3D raycasting engine
- **Game logic:** `UpdateGameObjects()` (internal) → `DoActor()` (rt_actor.h), `MoveDoors()` (rt_door.h), `ProcessElevators()` (rt_floor.h)
- **Input:** `UpdateClientControls()` → `PollControls()` (rt_playr.h); `PollKeyboard()` (internal) — debug/UI keys
- **Animation:** `AnimateWalls()` / `DoAnimatedMaskedWalls()` / `DoSprites()` (rt_stat.h) — sprite animation and masked-wall rendering
- **Pause & menu:** `PauseLoop()` (internal) → `ControlPanel()` (rt_menu.h) — in-game menu access

**Shutdown (QuitGame):**
- **Audio:** `MU_FadeOut()` (music.h)
- **Debug output:** `PrintMapStats()` / `PrintTileStats()` (rt_debug.h)
- **Subsystem teardown:** `ShutDown()` (rt_error.h) — unified shutdown for keyboard, timer, sound, memory
- **Exit:** `exit(0)` — terminate process

## Design Patterns & Rationale

**1. State Machine (exit_t enum)**
- **Pattern:** State pattern; playstate is a global flag that controls which branch of GameLoop's switch statement executes
- **Why:** Allows clear separation of menu, gameplay, death, level completion, and cinematic logic without deeply nested conditionals
- **Tradeoff:** Global state makes debugging hard; no explicit state transition validation; state changes scattered across multiple functions

**2. Initialization Ordering (rigid, sequential)**
- **Pattern:** Layered initialization (memory → input → audio/video → game state → menu)
- **Why:** Some subsystems depend on others being ready (e.g., audio can't initialize before memory manager exists; game state can't load before WAD system is up)
- **Tradeoff:** Late-discovered initialization errors are hard to recover from; no rollback mechanism if a subsystem fails mid-initialization

**3. Global Configuration & Feature Flags**
- **Pattern:** Compile-time (#if SHAREWARE, #if DEVELOPMENT) and runtime (tedlevel, warp, NoSound) feature gates
- **Why:** Controls product variants (shareware vs. registered), development/debug modes, and command-line overrides
- **Tradeoff:** Scattered boolean flags instead of unified configuration object; some flags only checked at startup, others dynamically

**4. Command-Line Parsing (early binding)**
- **Pattern:** Parse argv into global flags at startup; these flags influence initialization flow
- **Why:** Allows automated testing (TEDLEVEL, NOSOUND), demo playback (WARP), and network setup (net, numplayers)
- **Tradeoff:** No dynamic config reload; warp/tedlevel only work on startup, not mid-game

**5. Frame-Rate Independent Gameplay**
- **Pattern:** `PlayLoop()` calls `UpdateGameObjects()` in a loop while oldpolltime < oldtime, decoupling game logic ticks from render frames
- **Why:** Prevents frame-rate dependency bugs; allows rendering at different rates than game updates
- **Tradeoff:** Introduces timing complexity; requires careful sync checking in networked games

## Data Flow Through This File

```
Command-line args
    ↓ CheckCommandLineParameters()
    ↓ (sets global flags: tedlevel, warp, turbo, NoSound, etc.)
    ↓
Initialize Subsystems
    ├─ Z_Init() → memory allocator ready
    ├─ IN_Startup() → keyboard/input ready
    ├─ ReadConfig() / ReadSETUPFiles() → persistent settings loaded
    ├─ SetupWads() → WAD lumps indexed
    ├─ BuildTables() → lookup tables (sin, tan, gamma, etc.)
    ├─ MU_Startup() / SD_SetupFXCard() → audio hardware initialized
    └─ I_StartupTimer() / I_StartupKeyboard() → interrupt handlers installed
    ↓
playstate = ex_titles
    ↓
GameLoop()
    ├─ State branches (ex_titles → ex_resetgame → ex_stillplaying → ex_died → ex_titles)
    │
    ├─ ex_stillplaying:
    │   ↓
    │   PlayLoop()
    │       ├─ PollKeyboard() → interpret player input, update gamestate.autorun, detail level, etc.
    │       ├─ UpdateGameObjects() → advance actor positions, process triggers, check battle conditions
    │       ├─ ThreeDRefresh() → raycasting 3D render, fill framebuffer
    │       └─ FlipPage() → swap video buffers (VGA page flip)
    │
    └─ (other states: load cinematics, show menus, record/play demos, handle death/completion)
    ↓
QuitGame()
    ├─ MU_FadeOut()
    ├─ ShutDown() → close keyboard, timer, sound, free memory
    └─ exit(0)
```

**Key global state threaded through:**
- `gamestate` (global) — persistent across levels; includes health, score, weapon, difficulty, battle mode
- `playstate` (global) — current state machine branch
- `consoleplayer`, `numplayers` — player index and count (for single-player or networked multiplayer)
- `locplayerstate` → `&PLAYERSTATE[consoleplayer]` — player-specific state (position, velocity, health, ammo, keys)
- `Keyboard[]` / `LastScan` — input polling results (read by PollKeyboard, set by input ISR)
- Audio globals (`MUvolume`, `FXvolume`, `MusicMode`, `FXMode`) — volume and hardware state

## Learning Notes

**What developers studying this engine learn:**

1. **Frame-rate decoupling patterns**: The `oldtime` / `oldpolltime` timing loop is a practical approach to separating game updates from render frames, common in 90s engines before fixed timesteps became standard.

2. **DOS/VGA-era constraints**: Interrupt handlers (ISR), page flipping, mode X graphics, and COM port networking reflect pre-modern game dev. Modern engines abstract hardware completely; this one is tightly bound to it.

3. **Global state coordination**: Rather than dependency injection or event systems, rt_main.c coordinates subsystems by setting/checking global flags. Functional but brittle.

4. **State machine simplicity**: The switch-on-playstate pattern is readable but doesn't scale; each state has its own initialization and cleanup scattered across functions. A state stack or hierarchical FSM would be more flexible.

5. **Conditional compilation for products**: The SHAREWARE / REGISTERED / DELUXE branching shows how Apogee shipped multiple product variants from a single codebase—a common 90s practice now mostly replaced by runtime feature flags or separate builds.

6. **Graceful degradation**: Sound/music failures don't crash the game; instead, they set mode flags and prompt the user to run SNDSETUP. This is good UX for DOS-era hardware variability.

**Idiomatic to this engine / era:**
- No event bus, observer pattern, or message passing—direct function calls and globals
- Heavy use of compile-time constants to control behavior (SHAREWARE, DEVELOPMENT, WHEREAMI)
- ISR-based timing and input (interrupt handlers installed, not polled)
- Direct VGA hardware access (mode X, page flipping) instead of abstracted rendering
- Manual memory management (Z_Init with explicit tag-based cleanup)
- WAD lumps as asset database (used by Doom, Doom II, ROTT)

## Potential Issues

1. **Unprotected global state in multiplayer**: `playstate`, `gamestate`, etc. are modified by PlayLoop and UpdateGameObjects without apparent synchronization. If networked clients diverge, there's minimal validation (`CheckForSyncCheck()` exists but is sparse).

2. **Initialization order fragility**: If a subsystem (e.g., audio) fails partway through initialization, the code prints warnings but continues. This can leave the engine in a partially-initialized state, causing crashes later. A proper rollback/cleanup mechanism would be safer.

3. **Memory leaks on quit after failed load**: If Z_FreeTags() is called but some subsystems weren't fully initialized, dangling pointers are possible (though the tag-based allocator may mitigate this).

4. **Hard-coded paths and filenames**: ROTT0000.LBM, SNDSETUP are searched in implicit directories; no asset path configuration exists, making it fragile in non-standard installs.

5. **Race condition in demo recording**: `demorecord` and `demoplayback` are globals that can change mid-frame if CheckCommandLineParameters is re-invoked or if network desync occurs (unclear if this is possible).

---

**Word count: ~1400 tokens**
