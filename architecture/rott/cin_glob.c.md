# rott/cin_glob.c

## File Purpose
Provides a simple abstraction layer for controlling cinematic/cutscene playback. Wraps timing and input functions to allow cinematics to synchronize with the engine's tick system and detect user-initiated abort/skip requests.

## Core Responsibilities
- Synchronize cinematic playback with engine timing via tick counting
- Query elapsed cinematic time for audio/visual synchronization
- Detect user input indicating cinematic should be skipped/aborted
- Clear input acknowledgment state after processing abort requests

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### CinematicDelay
- **Signature:** `void CinematicDelay(void)`
- **Purpose:** Synchronize cinematic frame timing with engine tick system
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Calls `CalcTics()`, which updates global timing counters
- **Calls:** `CalcTics()` (defined in rt_draw.c)
- **Notes:** Typically called once per cinematic frame loop; tick-based instead of time-based for deterministic playback

### GetCinematicTime
- **Signature:** `int GetCinematicTime(void)`
- **Purpose:** Retrieve elapsed cinematic time as tick count for synchronization
- **Inputs:** None
- **Outputs/Return:** `int` – current value of global `ticcount`
- **Side effects:** None (read-only)
- **Calls:** None
- **Notes:** Used to sync audio playback or animation state to elapsed cinematic time; `ticcount` is a global extern from rt_draw.c

### CinematicAbort
- **Signature:** `boolean CinematicAbort(void)`
- **Purpose:** Detect whether user has pressed a key to skip/abort the cinematic
- **Inputs:** None
- **Outputs/Return:** `boolean` – TRUE if user input detected, FALSE otherwise
- **Side effects:** None (read-only status check)
- **Calls:** `IN_CheckAck()` (defined in rt_in.c)
- **Notes:** "Ack" = acknowledgment; typically checks for key press or controller input

### ClearCinematicAbort
- **Signature:** `void ClearCinematicAbort(void)`
- **Purpose:** Reset the input acknowledgment flag after processing an abort request
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Clears internal input state in rt_in.c
- **Calls:** `IN_StartAck()` (defined in rt_in.c)
- **Notes:** Must be called after `CinematicAbort()` returns TRUE to prevent repeated triggers

## Control Flow Notes
This module is part of the cinematic playback loop. Typical sequence:
1. Cinematic render/play loop repeatedly calls `CinematicDelay()` to advance time
2. Loop queries `GetCinematicTime()` to sync audio/animations
3. Loop checks `CinematicAbort()` to detect skip request
4. If skip requested, calls `ClearCinematicAbort()` to reset for next cinematic

Used during intro sequences, cutscenes, and credits (referenced in rt_draw.h: `ApogeeTitle()`, `DopefishTitle()`, `DoEndCinematic()`).

## External Dependencies
- **rt_draw.h / rt_draw.c:** `CalcTics()` function; `ticcount` global variable (extern int)
- **rt_in.h / rt_in.c:** `IN_CheckAck()`, `IN_StartAck()` functions
- **cin_glob.h:** Function declarations
- **memcheck.h:** Memory debugging utility (passive inclusion)
