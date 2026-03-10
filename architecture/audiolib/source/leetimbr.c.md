# audiolib/source/leetimbr.c

## File Purpose
Defines a lookup table of AdLib FM synthesizer instrument timbres/patches. The `ADLIB_TimbreBank` array contains 256 pre-configured instrument definitions that map to standard AdLib instrument slots used during gameplay audio synthesis.

## Core Responsibilities
- Store immutable timbre/instrument configuration data for AdLib FM synthesis
- Provide a 256-entry lookup table for instrument patch definitions
- Define synthesis parameters (envelope, waveform, feedback) for each instrument slot

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| TIMBRE | struct | Encapsulates an AdLib instrument configuration with modulator/carrier parameters |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| ADLIB_TimbreBank | TIMBRE[256] | global | Master lookup table of 256 pre-configured AdLib FM instrument definitions |

## Key Functions / Methods
None.

## Data Structure Details: TIMBRE

The `TIMBRE` struct contains two sets of synthesis parameters (one per oscillator):
- **SAVEK[2]**: Key Scale Level parameters (controls output level vs. note frequency)
- **Level[2]**: Operator output levels (amplitude envelope)
- **Env1[2]**: Envelope 1 attack/decay rates
- **Env2[2]**: Envelope 2 sustain/release parameters
- **Wave[2]**: Waveform select (sine, half-sine, abs-sine, pulse-sine)
- **Feedback**: Operator feedback amount (0–15, enabling FM modulation)
- **Transpose**: Pitch transpose in semitones (signed)
- **Velocity**: Note velocity sensitivity (0=fixed, positive=scaled)

## Control Flow Notes
This is a static data file with no control flow. The `ADLIB_TimbreBank` array is referenced at runtime by audio playback functions (elsewhere in audiolib) to configure FM synthesizer parameters when playing notes.

## External Dependencies
- None; self-contained data definitions only.

## Notes
- The struct is typedef'd but never instantiated directly; only `ADLIB_TimbreBank` is defined globally.
- Many entries repeat identical configurations (e.g., indices 2–10, 12–13, etc.), suggesting template/default instruments.
- Some entries (indices ~190–256) contain non-zero Velocity and Transpose fields, indicating percussion or special drum patch variants.
- Designed for the Yamaha OPL2 AdLib chip instruction set (classic 1989 FM synthesis hardware).
