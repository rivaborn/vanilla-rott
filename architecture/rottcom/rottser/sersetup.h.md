# rottcom/rottser/sersetup.h

## File Purpose
Header file declaring the interface for serial game setup and shutdown in ROTT's multiplayer/modem system. Exposes global state flags and entry point functions for initializing networked games.

## Core Responsibilities
- Declare external variables controlling modem and statistics display modes
- Export setup and shutdown functions for serial/networked game sessions
- Provide minimal interface to multiplayer game initialization subsystem

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `usemodem` | `boolean` | extern | Flag controlling modem/serial communication mode |
| `showstats` | `boolean` | extern | Flag controlling statistics display during serial game |

## Key Functions / Methods

### ShutDown
- Signature: `void ShutDown(void)`
- Purpose: Clean shutdown of serial game session and runtime
- Inputs: None
- Outputs/Return: None
- Side effects: Likely releases serial port, cleans up game state, terminates modem connection
- Calls: Not visible in this file (implementation in .c)
- Notes: Appears to be a global shutdown mechanism for the entire serial subsystem

### SetupSerialGame
- Signature: `void SetupSerialGame(void)`
- Purpose: Initialize and configure a serial/modem networked game session
- Inputs: None (uses global state via `usemodem`, `showstats`)
- Outputs/Return: None (configures global game state)
- Side effects: Initializes modem, serial port setup, game session parameters
- Calls: Not visible in this file (implementation in .c)
- Notes: Entry point for multiplayer game initialization; reads external flags to determine behavior

## Control Flow Notes
Part of the game initialization phase. Called during session setup when multiplayer/modem play is selected. `SetupSerialGame()` likely executes early in initialization, and `ShutDown()` during graceful termination.

## External Dependencies
- **Includes**: `global.h` (provides `boolean` typedef, utility function declarations)
- **External symbols**: Uses `boolean` type defined in `global.h`; actual implementation of `ShutDown()` and `SetupSerialGame()` defined elsewhere (likely in `sersetup.c`)
