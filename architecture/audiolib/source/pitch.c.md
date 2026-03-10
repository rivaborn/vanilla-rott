# audiolib/source/pitch.c

## File Purpose
Provides pitch scaling calculations for audio playback. Uses a precomputed lookup table to map MIDI-style pitch offsets (in cents) to fixed-point scale factors for playback rate adjustment. Includes DOS/DPMI memory locking to ensure deterministic real-time audio performance.

## Core Responsibilities
- Calculate fixed-point pitch scale factors from pitch offsets in cents
- Supply a precomputed 12-note × 25-detune lookup table for efficient pitch calculation
- Lock/unlock pitch code and data in physical memory for real-time audio without page swaps
- Support arbitrary pitch offsets by decomposing into octave, semitone, and detune components

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| PitchTable | `unsigned long[12][25]` | static | Precomputed hexadecimal fixed-point scale factors (0x10000 = 1.0×) for each semitone (rows 0–11) and detune step (columns 0–24). Values follow equal temperament. |

## Key Functions / Methods

### PITCH_GetScale
- **Signature:** `unsigned long PITCH_GetScale(int pitchoffset)`
- **Purpose:** Return a fixed-point scale factor for a given pitch offset in cents.
- **Inputs:** `pitchoffset` — pitch shift in cents (100 = one semitone, negative allowed).
- **Outputs/Return:** Fixed-point unsigned long; 0x10000 = 1.0× (no shift), 0x20000 = 2.0× (octave up), etc.
- **Side effects:** None (read-only).
- **Calls:** None (direct table lookup and bitwise arithmetic).
- **Notes:** 
  - Fast path for `pitchoffset == 0` returns 0x10000.
  - Decomposes input into octave shift, note (0–11), and detune (0–24) via modulo arithmetic.
  - Handles negative offsets by normalizing to 0–1199 cents range.
  - Octave shifts are implemented via left/right bit shifts on the lookup value (multiply/divide by powers of 2).

### PITCH_LockMemory
- **Signature:** `int PITCH_LockMemory(void)`
- **Purpose:** Lock PITCH_GetScale code and PitchTable data in physical memory for real-time, glitch-free audio.
- **Inputs:** None.
- **Outputs/Return:** `PITCH_Ok` (0) on success, `PITCH_Error` (−1) on failure.
- **Side effects:** Calls DPMI to lock memory regions; calls PITCH_UnlockMemory on failure.
- **Calls:** `DPMI_LockMemoryRegion(PITCH_LockStart, PITCH_LockEnd)`, `DPMI_Lock(PitchTable)`.
- **Notes:** Prevents OS page swaps during audio rendering. DOS/protected-mode specific.

### PITCH_UnlockMemory
- **Signature:** `void PITCH_UnlockMemory(void)`
- **Purpose:** Release locked memory regions back to the OS.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls DPMI to unlock.
- **Calls:** `DPMI_UnlockMemoryRegion(PITCH_LockStart, PITCH_LockEnd)`, `DPMI_Unlock(PitchTable)`.

## Control Flow Notes
Stateless utility module. `PITCH_GetScale` is expected to be called repeatedly during audio frame updates for pitch-shifted playback. Memory locking is performed once during audio subsystem initialization and unlocked on shutdown. The commented-out `PITCH_Init` suggests the table was precomputed offline; runtime initialization is not needed.

## External Dependencies
- **dpmi.h**: DOS DPMI memory locking (`DPMI_LockMemoryRegion`, `DPMI_UnlockMemoryRegion`, `DPMI_Lock`, `DPMI_Unlock`); error codes.
- **standard.h**: Standard macro definitions (included but unused in this file).
- **pitch.h**: Public interface declarations.
- **stdlib.h**: Standard C library (not directly used).
