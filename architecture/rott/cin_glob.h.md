# rott/cin_glob.h

## File Purpose
Public interface for cinematic (cutscene) timing and control. Provides functions to synchronize cinematic playback with the VBlank timer, query elapsed time, and handle user abort requests.

## Core Responsibilities
- Declare cinematic delay/synchronization function
- Expose cinematic elapsed time query
- Manage cinematic abort flag (check and clear)
- Define timing macro (`CLOCKSPEED`) based on VBlank counter

## Key Types / Data Structures
None.

## Global / File-Static State
None declared here (state maintained in implementation file, likely `cin_glob.c`).

## Key Functions / Methods

### CinematicDelay
- Signature: `void CinematicDelay(void)`
- Purpose: Pause execution until the next cinematic frame/VBlank tick; synchronizes cinematic playback to vertical blanking interval
- Inputs: None
- Outputs/Return: None
- Side effects: Blocks until timer interrupt; advances internal cinematic frame counter
- Calls: Not visible from this file
- Notes: Uses `CLOCKSPEED` (defined as `VBLCOUNTER` = 35 Hz from isr.h)

### GetCinematicTime
- Signature: `int GetCinematicTime(void)`
- Purpose: Return elapsed time since cinematic started, in timer ticks or milliseconds
- Inputs: None
- Outputs/Return: Integer time value (likely in VBlank ticks or centiseconds)
- Side effects: None (query only)
- Calls: Not visible from this file
- Notes: Likely reads from ISR timer state

### CinematicAbort
- Signature: `boolean CinematicAbort(void)`
- Purpose: Check whether the cinematic playback should be aborted (e.g., user skipped)
- Inputs: None
- Outputs/Return: `boolean` (true if abort requested)
- Side effects: None (query only)
- Calls: Not visible from this file
- Notes: Likely reads from a volatile flag set by input handling

### ClearCinematicAbort
- Signature: `void ClearCinematicAbort(void)`
- Purpose: Reset the abort flag after processing an abort request
- Inputs: None
- Outputs/Return: None
- Side effects: Clears internal abort flag
- Calls: Not visible from this file
- Notes: Should be called after handling cinematic skip

## Control Flow Notes
Integrates with the interrupt-driven timer subsystem (ISR, VBlank at 35 Hz). Likely used during cinematic/cutscene initialization and frame loop: call `CinematicDelay()` each frame, check `CinematicAbort()` for user input, and read `GetCinematicTime()` for audio/animation sync. The `CLOCKSPEED` macro ties cinematics to the display refresh.

## External Dependencies
- **rt_def.h** — provides `boolean` type, general engine constants
- **rt_util.h** — included (purpose not immediately apparent from declarations here)
- **isr.h** — provides `VBLCOUNTER` (35 Hz timer tick constant); declares ISR state (`ticcount`, keyboard queue)
- **<time.h>** — standard C time header (included but usage not inferable here)
