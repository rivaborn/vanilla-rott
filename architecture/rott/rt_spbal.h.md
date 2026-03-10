# rott/rt_spbal.h

## File Purpose
Public interface header for SpaceBall input device handling. Declares functions to initialize, poll, and shut down the SpaceBall device for reading button and positional input during gameplay.

## Core Responsibilities
- Initialize and open the SpaceBall device
- Shut down and close the SpaceBall device
- Poll SpaceBall input state each frame
- Query current SpaceBall button states

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### GetSpaceBallButtons
- Signature: `unsigned GetSpaceBallButtons(void)`
- Purpose: Query the current state of SpaceBall buttons
- Inputs: None
- Outputs/Return: Unsigned integer representing button state (bitmask)
- Side effects: None (read-only query)
- Calls: Not inferable from this file
- Notes: Likely called during input polling to detect button presses

### OpenSpaceBall
- Signature: `void OpenSpaceBall(void)`
- Purpose: Initialize and open the SpaceBall input device
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes SpaceBall hardware/driver
- Calls: Not inferable from this file
- Notes: Likely called during engine initialization

### CloseSpaceBall
- Signature: `void CloseSpaceBall(void)`
- Purpose: Shut down and close the SpaceBall input device
- Inputs: None
- Outputs/Return: None
- Side effects: Releases SpaceBall hardware resources
- Calls: Not inferable from this file
- Notes: Likely called during engine shutdown

### PollSpaceBall
- Signature: `void PollSpaceBall(void)`
- Purpose: Update SpaceBall input state from device
- Inputs: None
- Outputs/Return: None
- Side effects: Updates internal SpaceBall state
- Calls: Not inferable from this file
- Notes: Likely called once per frame during input polling phase

## Control Flow Notes
Fits into engine lifecycle: `OpenSpaceBall()` during init, `PollSpaceBall()` every frame in the input polling phase, `GetSpaceBallButtons()` during input handling, and `CloseSpaceBall()` during shutdown.

## External Dependencies
- None (header-only declarations)
