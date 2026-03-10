# rott/rt_rand.h

## File Purpose
Public interface for the random number generator system. Declares initialization, seeding, and RNG functions with conditional debug logging support controlled by the `RANDOMTEST` compile flag.

## Core Responsibilities
- Declare RNG initialization and seed management functions
- Provide `GameRNG()` and `RNG()` function wrappers with macro aliases
- Support debug mode (RANDOMTEST) that logs RNG calls with string labels and values
- Expose RNG state index management (get/set)
- Abstract production vs. debug signatures behind preprocessor conditionals

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### InitializeRNG
- Signature: `void InitializeRNG(void)`
- Purpose: Initialize the random number generator system
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up internal RNG state
- Calls: Not visible in header
- Notes: Must be called during engine startup before any RNG calls

### GetRandomSeed
- Signature: `int GetRandomSeed(void)`
- Purpose: Retrieve the current seed value used by the RNG
- Inputs: None
- Outputs/Return: `int` seed value
- Side effects: None
- Calls: Not visible in header

### GameRNG / GameRandomNumber
- Signature: 
  - (RANDOMTEST=1): `int GameRNG(char *string, int val)`
  - (RANDOMTEST=0): `int GameRNG(void)`
- Purpose: Generate a game-specific random number; optionally log with debug string
- Inputs: (debug mode) `string` label, `val` associated value
- Outputs/Return: `int` random number
- Side effects: Advances RNG state
- Calls: Not visible in header
- Notes: `GameRandomNumber(string,val)` macro provides unified interface; signature varies by compile mode

### RNG / RandomNumber
- Signature:
  - (RANDOMTEST=1): `int RNG(char *string, int val)`
  - (RANDOMTEST=0): `int RNG(void)`
- Purpose: Generate a general random number; optionally log with debug string
- Inputs: (debug mode) `string` label, `val` associated value
- Outputs/Return: `int` random number
- Side effects: Advances RNG state
- Calls: Not visible in header
- Notes: `RandomNumber(string,val)` macro provides unified interface; signature varies by compile mode

### SetRNGindex / GetRNGindex
- Signatures: `void SetRNGindex(int i)` / `int GetRNGindex(void)`
- Purpose: Manage internal RNG sequence position for state save/restore
- Inputs: (Set only) `int i` index value
- Outputs/Return: (Get only) `int` current index
- Side effects: Set modifies RNG state
- Calls: Not visible in header

## Control Flow Notes
Header-only interface. Initialization expected at engine startup (`InitializeRNG`); RNG calls issued during gameplay updates/events. Index save/restore likely used in save-game or replay systems.

## External Dependencies
- `develop.h` — provides `RANDOMTEST` and `RANDOMTEST` compile flags
