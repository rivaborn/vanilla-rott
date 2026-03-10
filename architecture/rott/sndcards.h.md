# rott/sndcards.h

## File Purpose
Header file defining enumerated sound card types supported by the game engine. Provides a centralized list of hardware sound devices that can be initialized and used for audio output, along with a version identifier for the audio subsystem.

## Core Responsibilities
- Define enumeration of supported sound cards (SoundBlaster, Adlib, GenMidi, SoundCanvas, etc.)
- Provide version identifier for the audio subsystem
- Serve as interface for audio initialization and device selection elsewhere in codebase

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `soundcardnames` | enum | Enumeration of all supported sound output devices (SoundBlaster, Adlib, GenMidi, SoundCanvas, Awe32, UltraSound, PC, etc.); includes sentinel value `NumSoundCards` for array sizing |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ASS_VERSION_STRING` | const char* (macro) | global | Version identifier for audio subsystem ("1.04") |

## Key Functions / Methods
None. This is a pure header file containing only type definitions.

## Control Flow Notes
This file does not participate in control flow. It is a definitions-only header typically `#include`d by audio initialization and device detection routines. The `soundcardnames` enum likely drives conditional logic in sound card detection or initialization code elsewhere.

## External Dependencies
- Standard C preprocessor (for `#define`, `#ifndef` guards)
- No explicit includes; used by external modules for type definitions

## Notes
- Enum includes a commented-out `ASS_NoSound` entry, suggesting legacy or removed sound-disabled mode
- `NumSoundCards` sentinel value enables iteration over all card types
- Reflects early-to-mid 1990s sound hardware support (SoundBlaster, Adlib, Roland SoundCanvas, Gravis UltraSound)
