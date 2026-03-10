# rott/rt_debug.h

## File Purpose
Header file declaring debug and cheat code subsystem entry points. Provides an interface for managing cheat codes, processing debug input, and controlling demo playback within the game engine.

## Core Responsibilities
- Declare cheat code initialization/reset function
- Declare debug key input handler
- Declare debug status check function
- Declare demo termination function

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### ResetCheatCodes
- Signature: `void ResetCheatCodes(void)`
- Purpose: Reset or initialize the cheat code system to a clean state
- Inputs: None
- Outputs/Return: None
- Side effects: Resets cheat code state (likely clears flags or internal counters)
- Calls: Not inferable from this file
- Notes: Likely called on game startup or level reset

### DebugKeys
- Signature: `int DebugKeys(void)`
- Purpose: Process debug-related keyboard input and return result
- Inputs: None (reads from input system)
- Outputs/Return: Integer status/result code
- Side effects: May modify debug state or trigger cheat code activation
- Calls: Not inferable from this file
- Notes: Return value suggests success/failure or cheat code ID; likely called once per frame during input phase

### CheckDebug
- Signature: `void CheckDebug(void)`
- Purpose: Perform debug status checks or conditional debug behavior
- Inputs: None
- Outputs/Return: None
- Side effects: May log debug output or modify debug state
- Calls: Not inferable from this file
- Notes: Likely called periodically during frame update

### EndDemo
- Signature: `void EndDemo(void)`
- Purpose: Terminate the current demo playback sequence
- Inputs: None
- Outputs/Return: None
- Side effects: Stops demo recording/playback and returns to normal gameplay
- Calls: Not inferable from this file
- Notes: Likely called by debug input handlers or on demo completion

## Control Flow Notes
Functions are typical debug subsystem entry points, likely called from the main game loop's input/update phase. `DebugKeys()` runs each frame to poll for cheat activation; `CheckDebug()` performs periodic debug checks. `EndDemo()` is event-driven.

## External Dependencies
- No explicit includes shown (standard C header convention)
- Implementation in `rt_debug.c` (inferred from file comment)
- Assumes external input and demo systems exist
