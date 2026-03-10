# rott/rt_rand.c

## File Purpose
Implements a deterministic pseudo-random number generator using a pre-computed lookup table of 2048 values. Provides two independent RNG streams for game logic and other subsystems (e.g., sound), with support for debug logging and state inspection for replay/record functionality.

## Core Responsibilities
- Initialize RNG system at engine startup with time-based seeds
- Provide two independent random value streams (GameRNG and RNG)
- Manage circular-buffer indices for deterministic playback
- Support optional debug mode that logs all RNG calls with source information
- Allow external query and manual manipulation of RNG indices for testing/replay

## Key Types / Data Structures
None (uses pre-computed RandomTable from `_rt_rand.h`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| rndindex | int | static | Current position in RandomTable for game RNG stream |
| sndindex | int | static | Current position in RandomTable for secondary RNG stream |

## Key Functions / Methods

### GetRandomSeed
- Signature: `int GetRandomSeed(void)`
- Purpose: Generate initial seed value from system time
- Inputs: None
- Outputs/Return: int (seed in range [0, SIZE_OF_RANDOM_TABLE))
- Side effects: Calls `time(NULL)` from libc
- Calls: time (libc)
- Notes: Uses modulo to fit time value into valid table range

### InitializeRNG
- Signature: `void InitializeRNG(void)`
- Purpose: Initialize both RNG streams at engine startup
- Inputs: None
- Outputs/Return: void
- Side effects: Calls SetRNGindex() and GetRandomSeed() twice to initialize rndindex and sndindex
- Calls: SetRNGindex(), GetRandomSeed()
- Notes: Called once during engine initialization

### SetRNGindex
- Signature: `void SetRNGindex(int i)`
- Purpose: Manually set game RNG index (for replay/testing)
- Inputs: int i – new index value
- Outputs/Return: void
- Side effects: Modifies rndindex; calls SoftError() for logging
- Calls: SoftError()
- Notes: SoftError call always active despite commented #if guard; used for record/replay state control

### GetRNGindex
- Signature: `int GetRNGindex(void)`
- Purpose: Query current game RNG index
- Inputs: None
- Outputs/Return: int – current rndindex
- Side effects: None
- Calls: None

### GameRNG
- Signature (production): `int GameRNG(void)`
- Signature (debug): `int GameRNG(char *string, int val)`
- Purpose: Advance game RNG stream and return next random value
- Inputs: (debug only) char *string – call site identifier, int val – context
- Outputs/Return: int – next value from RandomTable
- Side effects: Increments rndindex with circular wrapping; optional SoftError logging (debug build only)
- Calls: SoftError() (RANDOMTEST==1 only)
- Notes: Two implementations via conditional compilation (RANDOMTEST flag). Uses bitwise AND `(rndindex+1)&(SIZE_OF_RANDOM_TABLE-1)` for power-of-2 circular buffer

### RNG
- Signature (production): `int RNG(void)`
- Signature (debug): `int RNG(char *string, int val)`
- Purpose: Advance secondary RNG stream and return next random value
- Inputs: (debug only) char *string – call site identifier, int val – context
- Outputs/Return: int – next value from RandomTable
- Side effects: Increments sndindex with circular wrapping; optional SoftError logging (commented out in debug variant)
- Calls: SoftError() (commented; would log only if enabled)
- Notes: Independent stream from GameRNG. Same circular-buffer mechanism. Debug SoftError intentionally commented out.

## Control Flow Notes
- **Startup**: InitializeRNG() seeds both indices from system time
- **Runtime**: GameRNG() and RNG() called throughout engine to pull deterministic pseudo-random values
- **Debug/Replay**: SetRNGindex() and GetRNGindex() enable state inspection and manipulation for record-and-replay or debugging
- Exact integration points (frame/update/render phases) not inferable from this file alone

## External Dependencies
- **System includes**: `<time.h>` (time function)
- **Local headers**: `rt_def.h`, `_rt_rand.h` (RandomTable, SIZE_OF_RANDOM_TABLE), `rt_rand.h`, `develop.h` (RANDOMTEST, DEVELOPMENT flags), `rt_util.h` (SoftError macro), `memcheck.h`
- **Defined elsewhere**: RandomTable (2048 pre-computed byte values in _rt_rand.h), SoftError (macro from rt_util.h), SIZE_OF_RANDOM_TABLE constant
