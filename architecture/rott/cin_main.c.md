# rott/cin_main.c

## File Purpose
Core cinematic playback engine for ROTT. Orchestrates loading, parsing, and executing cinematic scripts; manages timing, frame updates, and rendering for in-game cinematics.

## Core Responsibilities
- Load and parse cinematic script files from disk or WAD lumps
- Manage cinematic timing, frame deltas, and tics-per-frame calculation
- Coordinate startup/shutdown of cinematic subsystems (events, actors, effects)
- Execute main cinematic playback loop with event/actor updates
- Profile rendering performance to calibrate frame timing

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cinematicdone` | boolean | global | Signals cinematic playback completion |
| `cinematictime` | int | static | Current elapsed cinematic time (tics) |
| `cinematictics` | int | static | Frame delta in tics (calculated per frame) |
| `cinematictictime` | int | static | Previous frame's timestamp for delta calculation |
| `profiletics` | int | static | Cached profiling result (time per ProfileDisplay call) |

## Key Functions / Methods

### PlayMovie
- **Signature:** `void PlayMovie(char *name, boolean uselumpy)`
- **Purpose:** Main entry point; executes complete cinematic playback from script load to shutdown
- **Inputs:** `name` (cinematic basename), `uselumpy` (load from WAD vs. disk)
- **Outputs/Return:** None
- **Side effects:** Initializes cinematic subsystems, loads script, modifies global cinematictime/cinematictics, updates events/actors, renders output
- **Calls:** StartupCinematic, GrabCinematicScript, PrecacheCinematic, GetCinematicTics, CinematicAbort, UpdateCinematicEvents, UpdateCinematicActors, DrawCinematicActors, ShutdownCinematic
- **Notes:** Loops until `cinematicdone` is true or abort flag set; increments cinematictime once per tic

### GrabCinematicScript
- **Signature:** `void GrabCinematicScript(char const *basename, boolean uselumpy)`
- **Purpose:** High-level script loading wrapper; chooses between disk and WAD loading, then parses
- **Inputs:** `basename` (filename without extension), `uselumpy` (use WAD system)
- **Outputs/Return:** None
- **Side effects:** Loads script into memory, initializes script parsing state
- **Calls:** LoadScriptFile, CacheScriptFile, ParseCinematicScript
- **Notes:** Appends ".ms" extension to basename

### ParseCinematicScript
- **Signature:** `void ParseCinematicScript(void)`
- **Purpose:** Parse cinematic event tokens from loaded script buffer; accumulate time offsets and dispatch events
- **Inputs:** None (reads from global `script_p`, `scriptend_p`, `endofscript`)
- **Outputs/Return:** None
- **Side effects:** Consumes script_p pointer, modifies parsing state
- **Calls:** GetToken, ParseNum, ParseEvent
- **Notes:** Accumulates relative time values; ParseEvent called with absolute time

### CacheScriptFile
- **Signature:** `void CacheScriptFile(char *filename)`
- **Purpose:** Load script from WAD lump into cached memory
- **Inputs:** `filename` (lump name, no extension)
- **Outputs/Return:** None
- **Side effects:** Allocates PU_CACHE tagged memory, sets up script_p/scriptend_p/scriptline globals
- **Calls:** W_GetNumForName, W_CacheLumpNum, W_LumpLength
- **Notes:** Assumes lump exists; PU_CACHE tagged memory can be purged by memory manager

### GetCinematicTics
- **Signature:** `void GetCinematicTics(void)`
- **Purpose:** Calculate frame delta in tics since last frame; uses profiled performance metric
- **Inputs:** None (reads global cinematictictime, profiletics)
- **Outputs/Return:** None
- **Side effects:** Waits for timer to advance, updates cinematictics and cinematictictime
- **Calls:** GetCinematicTime
- **Notes:** Busy-waits for timer change; sets cinematictics to cached profiletics (not actual delta)

### StartupCinematic
- **Signature:** `void StartupCinematic(void)`
- **Purpose:** Initialize cinematic subsystems and state prior to playback
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initializes events and actors, resets timing, clears abort flag, profiles machine
- **Calls:** StartupEvents, StartupCinematicActors, ClearCinematicAbort, GetCinematicTics, ProfileMachine
- **Notes:** Call before loading script

### ShutdownCinematic
- **Signature:** `void ShutdownCinematic(void)`
- **Purpose:** Clean up cinematic subsystems
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Releases event and actor resources
- **Calls:** ShutdownEvents, ShutdownCinematicActors
- **Notes:** Call after playback complete

### ProfileMachine
- **Signature:** `void ProfileMachine(void)`
- **Purpose:** Measure rendering performance to establish frame time; calibrates cinematictics
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls ProfileDisplay 4×, caches result in profiletics; only runs once
- **Calls:** GetCinematicTime, ProfileDisplay
- **Notes:** Early return if profiletics already set; ensures profiletics ≥ 1; runs during startup

## Control Flow Notes
**Initialization** → StartupCinematic (profiles, clears state) → **Script Load** → GrabCinematicScript (loads + parses events) → PrecacheCinematic (asset precache) → **Playback Loop** → GetCinematicTics (calc frame delta) → UpdateCinematicEvents/UpdateCinematicActors → DrawCinematicActors → repeat until abort → **Shutdown** → ShutdownCinematic.

## External Dependencies
- **Cinematic subsystems:** cin_glob.h (timing), cin_actr.h, cin_evnt.h, cin_efct.h (update/render)
- **Scripting:** scriplib.h (script parsing globals and functions)
- **WAD system:** w_wad.h (W_GetNumForName, W_CacheLumpNum, W_LumpLength)
- **Graphics:** modexlib.h, lumpy.h (ProfileDisplay)
- **Memory:** z_zone.h (PU_CACHE tag for lump caching)
- **Standard C:** stdio.h, stdlib.h, string.h, conio.h, dos.h, malloc.h, fcntl.h, io.h
