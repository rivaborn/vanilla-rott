# rott/cin_main.h

## File Purpose
Public API for the cinematic/movie playback system in ROTT. Declares functions to load and play movie scripts, manage cinematic timing, and track playback state through a global completion flag.

## Core Responsibilities
- Load movie script files into memory for playback
- Play loaded cinematic sequences with optional rendering modes
- Update cinematic timing state each frame
- Expose global completion status for cinematics
- Support "lumpy" rendering mode (likely palette-based or chunked rendering)

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cinematicdone` | boolean | global | Flag indicating cinematic playback has completed |

## Key Functions / Methods

### GrabMovieScript
- Signature: `void GrabMovieScript (char const *basename, boolean uselumpy);`
- Purpose: Load and parse a movie script file into memory
- Inputs: `basename` (script filename without path/extension); `uselumpy` (enable lumpy rendering mode)
- Outputs/Return: void (result stored in global cinematic state)
- Side effects: File I/O; modifies cinematic system state
- Calls: Not visible from header
- Notes: Uses `const char*` for basename (read-only), suggesting file path is not modified

### PlayMovie
- Signature: `void PlayMovie (char * name, boolean uselumpy);`
- Purpose: Execute/render a previously loaded movie script
- Inputs: `name` (movie identifier); `uselumpy` (rendering mode flag)
- Outputs/Return: void (produces video output)
- Side effects: Screen rendering; updates `cinematicdone` flag on completion
- Calls: Not visible from header
- Notes: Blocking playback loop; likely sets `cinematicdone` when finished

### GetCinematicTics
- Signature: `void GetCinematicTics ( void );`
- Purpose: Update cinematic frame timing on each game tick
- Inputs: none
- Outputs/Return: void (updates global timing state)
- Side effects: Advances cinematic clock/frame counter
- Calls: Not visible from header
- Notes: Called per-frame during active cinematic playback

## Control Flow Notes
Cinematic lifecycle: `GrabMovieScript` (init phase) → `PlayMovie` (intro/cutscene playback) → `GetCinematicTics` (per-frame tick) → `cinematicdone` polled to return to gameplay. The `uselumpy` flag likely optimizes rendering for hardware constraints or palette-based displays.

## External Dependencies
- `cin_glob.h` (local cinematic globals/macros)
- `rt_def.h`, `rt_util.h`, `isr.h` (engine core, included transitively)
- `<time.h>` (standard C timing)
- `boolean` type and `VBLCOUNTER` macro defined elsewhere in engine
