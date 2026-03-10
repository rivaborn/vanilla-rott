# audiolib/source/sndcards.h

## File Purpose
Header file defining enumerated types for supported audio device types in the sound library. Provides a centralized list of sound card identifiers and a version constant for the audio subsystem.

## Core Responsibilities
- Enumerate all supported sound card device types (Sound Blaster, Adlib, SoundCanvas, UltraSound, etc.)
- Define the library version string
- Provide shared type definitions for the audio subsystem initialization and device selection

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `soundcardnames` | enum | Enumeration of supported audio device types; used to identify which audio driver to initialize |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ASS_VERSION_STRING` | macro (string) | global | Version identifier for the audio library ("1.12") |

## Key Functions / Methods
None (header-only type definitions).

## Control Flow Notes
This is a header file used during initialization and device detection phases. Other modules include this to reference sound card type constants when selecting which audio driver to load or initialize. The `NumSoundCards` sentinel value provides a count for iterating over device types.

## External Dependencies
- Standard C preprocessor directives only (`#ifndef`, `#define`, `#endif`)
- No external symbol dependencies

**Notes:**
- One enum entry is commented out (`ASS_NoSound`), suggesting prior removal or conditional support
- Device enumeration reflects late 1980s/early 1990s audio hardware (Sound Blaster, Adlib, UltraSound, etc.)
- PC and SoundScape entries may represent fallback/generic options
