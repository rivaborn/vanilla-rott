# rott/_rt_soun.h — Enhanced Analysis

## Architectural Role

This private header encapsulates the **sound instance management layer** of the game's audio subsystem, sitting between high-level game code (actors, weapons, UI) and the hardware-abstracted audio driver layer (Adlib, Blaster, GUS). It defines the data structures and priority discipline for the real-time sound queue, which is central to ROTT's multi-voice audio arbitration in a resource-constrained DOS environment.

## Key Cross-References

### Incoming (who depends on this file)
- **Sound system public API** (`rt_soun.c`, likely `rt_soun.h`): Implements queuing/dispatch logic using `sound_t`
- **Game entities** (actors, weapons, player controller): Call `SD_PlayIt()` after calculating angle/distance from listener
- **Audio driver selection layer**: References `soundtype` variable to pick between SOUNDTYPES (likely Adlib vs. Blaster/GUS modes)

### Outgoing (what this file depends on)
- **Globals**: `sounds[]` array (sound metadata/handles), `soundtype` variable (active driver)
- **Utilities**: `RandomNumber()` function (for `PitchOffset()` macro—adds variation to repeated sounds)
- **Hardware enum constants**: `USEADLIB` (= 255), `GUSMIDIINIFILE` (suggests runtime audio config)
- **Audio driver subsystem** (below): Adlib, GUS MIDI, Blaster drivers (referenced indirectly via driver selection)

## Design Patterns & Rationale

| Pattern | Why |
|---------|-----|
| **Private header + priority queue** | Limited simultaneous voices (DOS-era constraint); priority prevents low-importance sounds (footsteps, grenades) from blocking critical ones (boss roars, pickups). 17 levels allow fine-grained categorization. |
| **Bit flags (`SD_*`) for behavior** | Compact, cache-friendly representation of sound behavior (loop, overwrite, no-pitch-shift); avoids bloated bool fields. |
| **`prevhandle` / `prevdistance` tracking** | Enables **distance-based preemption**: if a closer sound wants to play at a filled priority level, evict the farther sound at same priority. Classic culling heuristic for 3D audio. |
| **`SOUNDTYPES=2` hardcoded array** | Supports simultaneous multi-driver audio (e.g., music on MIDI device, SFX on digital). Not dynamic; tied to hardware detection at init. |
| **Macro-based config** (`PitchOffset`, `SoundOffset`) | Inlines simple lookups; avoids function call overhead in tight audio loops. |

## Data Flow Through This File

```
Game Entity (actor, weapon, UI)
  → calls SD_PlayIt(sndnum, angle, distance, pitch)
  → [lookup: sounds[sndnum] → get sound metadata]
  → [check priority + distance against active sound_t queue]
  → [if no slot: preempt lower-priority or farther sound]
  → [retrieve handle via SoundOffset(sndnum) → snds[soundtype]]
  → [apply pitch variation via PitchOffset() → RandomNumber()]
  → [dispatch to audio driver with angle/distance parameters]
  → Audio driver (Blaster/Adlib/GUS) → speaker
```

**Key insight**: `sound_t` is the **instance record** of a currently-playing sound (not the sound asset). Each entry tracks the two hardware variants' handles, priority, and distance for culling.

## Learning Notes

### Idiomatic to 1990s DOS games
- **Fixed-size voice/sound queue**: Modern engines use dynamic pools; ROTT pre-allocates based on hardware limitations.
- **Distance culling instead of dynamic volume falloff**: Simplistic but efficient—if max voices full, drop farther sounds rather than dynamically mix volumes.
- **Separate MIDI and digital audio tracks**: Multi-device architecture (MIDI for music, digital for SFX) was necessary when sound cards had limited mixing capacity.
- **Enum-based priority** (not score-based): No runtime priority computation; values are baked in.

### Connections to modern engine concepts
- **Priority queue discipline** = modern audio managers still use priority (dialogue > music > ambient).
- **Instance tracking (sound_t)** ≈ modern voice object in Wwise/FMOD.
- **Distance-based preemption** ≈ simplified version of modern "auralization" (3D panning, HRTF filtering).
- **Bit flags** ≈ modern behavior modifiers (looping, priority adjustment per-instance).

## Potential Issues

1. **Hardcoded `SOUNDTYPES=2`**: If more audio driver types are supported elsewhere, this struct is inflexible. Adding a third driver would require recompiling all code that uses `sound_t`.

2. **`prevdistance` type unclear**: Field is `int` but distance may wrap or overflow; no bounds checking visible. Could lead to incorrect preemption logic.

3. **Missing voice count**: The `sound_t` array size is not defined here (likely in implementation). Unclear how many simultaneous sounds are supported—impacts priority level utility.
