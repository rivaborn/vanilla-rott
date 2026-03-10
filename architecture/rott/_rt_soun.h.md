# rott/_rt_soun.h

## File Purpose
Private header defining internal structures, flags, and constants for the Rise of the Triad sound system. Manages active sound instances, music data, and priority/behavior configuration for audio playback.

## Core Responsibilities
- Define `sound_t` structure for tracking active sound instances
- Define `song_t` structure for music/song metadata
- Enumerate 17 priority levels for sound playback arbitration
- Provide sound behavior flags (looping, overwrite, pitch shift, etc.)
- Map sound types to priority constants (e.g., explosions, weapon fire, enemies, UI)
- Declare sound playback and validation functions
- Expose tuning constants (distance shift, random pitch shift)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `sound_t` | struct | Active sound instance: two sound handle variants, flags, priority level, count, previous handle/distance for distance-based culling |
| `song_t` | struct | Music metadata: loop flag, song type, 9-char lump name, 40-char song name |
| (unnamed enum `sd_prio*`) | enum | 17 priority levels (17 down to 1); higher value = higher priority |
| `looptypes` | enum | Boolean-like: `loop_yes`, `loop_no` |

## Global / File-Static State
None.

## Key Functions / Methods

### SD_PlayIt
- Signature: `int SD_PlayIt(int sndnum, int angle, int distance, int pitch)`
- Purpose: Queue/play a sound with position and pitch parameters
- Inputs: sound index, angle (direction), distance (attenuation), pitch offset
- Outputs/Return: sound handle (int)
- Side effects: Updates active sound queue; may preempt lower-priority sounds
- Calls: Not visible in this file
- Notes: Called after distance/priority filtering

### SD_SoundOkay
- Signature: `boolean SD_SoundOkay(int sndnum)`
- Purpose: Validate that a sound exists and is loadable
- Inputs: sound index
- Outputs/Return: boolean (true if sound is okay)
- Side effects: None
- Calls: Not visible in this file
- Notes: Likely used before attempting playback

## Control Flow Notes
This is a private header; control flow is in the implementation file. Priority constants suggest a prioritized sound queue where lower-priority sounds are culled when slots are full. Distance-based attenuation and pitch shifting are supported. The `SoundOffset()` macro retrieves sound handles by type.

## External Dependencies
- `sounds[]` array (macro `SoundOffset` references; defined elsewhere)
- `soundtype` variable (used in `SoundOffset` macro)
- `RandomNumber()` function (used in `PitchOffset` macro; defined elsewhere)
- `USEADLIB` constant (= 255; likely device/driver enum)
- `GUSMIDIINIFILE` path reference

**Trivial helpers:**
- Macros: `PitchOffset()` (random pitch ±128), `SoundOffset(x)` (retrieve sound handle), `SD_DISTANCESHIFT`, `SD_RANDOMSHIFT` (tuning constants)
- Flags: `SD_OVERWRITE`, `SD_WRITE`, `SD_LOOP`, `SD_PITCHSHIFTOFF`, `SD_PLAYONCE` (5 bits for sound behavior)
