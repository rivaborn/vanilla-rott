# audiolib/source/pitch.h — Enhanced Analysis

## Architectural Role

The PITCH module serves as a **utility layer for real-time pitch calculations** used by the game's MIDI and sound card subsystems. It abstracts the math for pitch-offset-to-scale conversions, allowing higher-level audio drivers (like AL MIDI and sound card synthesizers) to request pitch transformations without managing the underlying data structures. The lock/unlock pair indicates this module manages **memory-resident lookup tables or precomputed data** needed for low-latency pitch operations during audio playback.

## Key Cross-References

### Incoming (who depends on this file)
Based on the architecture, these subsystems likely call PITCH functions:
- **AL MIDI subsystem** (`audiolib/source/al_midi.c`): `AL_CalcPitchInfo` and `AL_SetVoicePitch` suggest pitch calculation during note rendering
- **Sound card drivers** (ADLIBFX, Blaster, GUS, AWE32): Any synthesizer supporting pitch bend or transposition would invoke `PITCH_GetScale`
- **Initialization code**: Whoever initializes the audio system likely calls `PITCH_LockMemory` early on

### Outgoing (what this file depends on)
- No explicit dependencies visible in header (implementation in PITCH.C)
- Likely depends on: fixed-point math utilities, possibly lookup table data structures
- The commented-out `PITCH_Init()` suggests initialization was previously part of this module but is now handled elsewhere

## Design Patterns & Rationale

**Computation-to-Lookup Hybrid**
- `PITCH_GetScale()` returns a precomputed or cached value, avoiding runtime multiplication/division on every note event
- This reflects 1990s real-time audio constraints: lookup tables beat arithmetic for latency-critical paths

**Lifecycle Management (Lock/Unlock)**
- `PITCH_LockMemory()` / `PITCH_UnlockMemory()` protect pitch data from memory swapping
- Critical for DOS/early Windows where interrupt handlers must access locked memory
- Suggests PITCH data is accessed from audio interrupt handlers (synchronous real-time context)

**Minimal Public Surface**
- Only 3 functions exposed; implementation hidden in PITCH.C
- Reflects "sealed" utility module design: callers don't need to understand pitch math internals

## Data Flow Through This File

**Initialization Phase:**
1. Audio system startup calls `PITCH_LockMemory()` 
2. PITCH module allocates/protects lookup tables in locked memory
3. Returns success/failure code to caller

**Runtime Phase (per note event):**
1. MIDI or sound card driver receives a note with pitch offset
2. Calls `PITCH_GetScale(offset)` → retrieves precomputed scale factor (typically in fixed-point format)
3. Uses scale factor to modulate the note's pitch

**Shutdown Phase:**
1. Audio system calls `PITCH_UnlockMemory()`
2. PITCH frees locked memory resources

**Data at rest:** Likely a table of pitch scale multipliers indexed by pitch offset (-128 to +127, or similar MIDI range).

## Learning Notes

**What This File Teaches:**
- **Early 1990s DOS/Windows audio architecture**: Lock/unlock was essential for DMA transfers and interrupt handlers
- **Separation of concerns in audio libraries**: Math/lookup provision (PITCH) vs. hardware abstraction (sound card drivers)
- **Fixed-point optimization patterns**: Precomputed lookup tables beat runtime arithmetic in real-time systems

**Idiomatic to This Era vs. Modern Engines:**
- **Then:** Manual memory locking, precomputed lookup tables, interrupt-safe designs
- **Now:** Automatic memory management, SIMD vectorization, lock-free data structures, higher-level audio APIs

**Connections to Game Engine Concepts:**
- Analogous to **audio DSP graph nodes** in modern engines (each node transforms data)
- The lock/unlock pattern predates but anticipates **resource lifecycle management** (modern: RAII, smart pointers)

## Potential Issues

- **`PITCH_Init()` is commented out**: Initialization responsibility is unclear—was it moved? Could cause memory leaks if `PITCH_LockMemory()` is called without proper `PITCH_Init()` setup
- **Sparse error reporting**: `PITCH_GetScale()` returns `unsigned long` with no error indication—what if pitch offset is out of bounds? Assumes valid input from callers.
- **No visible return value semantics for lock/unlock**: Is return value success code or size? No error enum mapping visible.
