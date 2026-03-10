# audiolib/public/include/sndcards.h

## File Purpose
Header file defining enumerated types for sound card hardware supported by the audio library. Provides identifiers for various audio output devices used during runtime sound card initialization and selection. Contains version information for the audio subsystem.

## Core Responsibilities
- Enumerate all supported sound card hardware types (SoundBlaster, Adlib, UltraSound, etc.)
- Provide a standardized `soundcardnames` type for code referencing audio devices
- Define audio library version string (`ASS_VERSION_STRING`)
- Serve as the single source of truth for available audio device identifiers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `soundcardnames` | enum | Enumerated type listing all supported sound cards and a sentinel `NumSoundCards` value |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ASS_VERSION_STRING` | macro | global | Audio subsystem version identifier ("1.12") |

## Key Functions / Methods
None (header-only data definitions).

## Control Flow Notes
No control flow; this is a data-only header. The `soundcardnames` enum is used elsewhere in the audio library to index into lookup tables, control sound initialization logic, and route audio output to the appropriate driver.

## External Dependencies
- None (no includes or external symbols)

## Notes
- Commented-out `ASS_NoSound` enum value suggests earlier iteration included a "no sound" option
- `NumSoundCards` sentinel is used to determine enum bounds; currently evaluates to 12
- Card types span legacy hardware (Adlib, Sound Canvas) to MIDI synthesizers (Awe32, WaveBlaster), indicating broad DOS-era audio hardware support
