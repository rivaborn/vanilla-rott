# audiolib/public/include/fx_man.h
## File Purpose
Public API header for the FX_MAN sound effects management module. Provides high-level sound initialization, playback control, and mixing for the Apogee audiolib, supporting multiple sound card types and audio formats (VOC, WAV, raw PCM) with 3D positioning and effects.

## Core Responsibilities
- Initialize and configure sound cards (Sound Blaster variants, Pro Audio Spectrum, Adlib, etc.)
- Manage voice allocation and priority-based sound queuing
- Play audio samples in multiple formats with pitch, panning, and volume control
- Support looped and non-looped playback with callback notifications
- Implement 3D audio positioning (angle/distance)
- Provide reverb and spatial audio effects
- Handle audio recording and demand-feed playback
- Manage global mixer settings (volume, stereo reverse, reverb depth/delay)

## External Dependencies
- **Includes:** `sndcards.h` — defines sound card enum (SoundBlaster, Adlib, GenMidi, etc.)
- **Defined elsewhere:** Actual implementation in `FX_MAN.C`; hardware-level sound card I/O drivers; interrupt handlers and DMA management (not visible in header)

# audiolib/public/include/music.h
## File Purpose
Public header for the MUSIC.C module providing the interface for MIDI music playback and control in the game engine. Defines error codes, song timing structures, and function signatures for initializing, playing, controlling, and monitoring music playback across different sound cards and MIDI devices.

## Core Responsibilities
- Initialize/shutdown music system with specified sound card and I/O address
- Play, pause, continue, and stop MIDI songs with loop control
- Control global and per-MIDI-channel volume, including fade effects
- Seek/position songs by ticks, milliseconds, or measure/beat/tick notation
- Query song playback state (playing, position, length)
- Configure MIDI channel mapping and timbre banks
- Route MIDI channels to custom callback handlers
- Report errors via error codes and human-readable messages

## External Dependencies
- **sndcards.h**: Sound card type definitions (`soundcardnames` enum)
- **Defined elsewhere**: All function implementations (MUSIC.C), MIDI driver layer, sound card initialization

# audiolib/public/include/sndcards.h
## File Purpose
Header file defining enumerated types for sound card hardware supported by the audio library. Provides identifiers for various audio output devices used during runtime sound card initialization and selection. Contains version information for the audio subsystem.

## Core Responsibilities
- Enumerate all supported sound card hardware types (SoundBlaster, Adlib, UltraSound, etc.)
- Provide a standardized `soundcardnames` type for code referencing audio devices
- Define audio library version string (`ASS_VERSION_STRING`)
- Serve as the single source of truth for available audio device identifiers

## External Dependencies
- None (no includes or external symbols)


# audiolib/public/include/task_man.h
## File Purpose
Public header for a low-level timer-based task scheduler used in the audio library. Manages periodic task execution with priority levels and interrupt-driven dispatch, typical of DOS-era game engine design where timer interrupts drive frame-locked tasks.

## Core Responsibilities
- Define task scheduling primitives (creation, termination, rate adjustment)
- Provide task dispatch mechanism for interrupt handlers or main loop
- Manage task linked lists with priority and execution rates
- Track interrupt context to allow code reuse in interrupt and non-interrupt paths
- Lock/unlock memory for interrupt safety (DOS-specific)

## External Dependencies
- None (self-contained public header; implementation in audiolib).

# audiolib/public/include/usrhooks.c
## File Purpose
Provides wrapper functions for memory allocation and deallocation operations that the audio library requires. These functions are intentionally left public and modifiable so the calling program can customize or restrict memory operations as needed.

## Core Responsibilities
- Wrap standard memory allocation (malloc) with error handling and return code semantics
- Wrap standard memory deallocation (free) with validation and return code semantics
- Provide a customization point for the library caller to override memory management behavior
- Maintain interface compatibility through function prototypes in the header

## External Dependencies
- **stdlib.h** — standard C library (malloc, free)
- **usrhooks.h** — local header defining error codes and function prototypes

# audiolib/public/include/usrhooks.h
## File Purpose
Public header for user hook interface in the audio library. Defines memory management function prototypes that allow the calling program to provide custom allocators and deallocators, enabling the audio library to respect application-specific memory constraints.

## Core Responsibilities
- Define memory allocation hook (`USRHOOKS_GetMem`)
- Define memory deallocation hook (`USRHOOKS_FreeMem`)
- Define error codes for hook operation results

## External Dependencies
- Standard C (no external includes in this header)
- Implementations defined elsewhere (`USRHOOKS.C`)

# audiolib/public/pm/source/pm.c
## File Purpose
Command-line MIDI player utility for DOS environments. Loads and plays MIDI files with interactive playback control, supporting multiple sound card configurations and real-time position/tempo seeking via keyboard commands.

## Core Responsibilities
- Parse command-line arguments for sound card selection, MIDI file, timbre bank, and playback position
- Initialize and manage music subsystem with configurable hardware addresses
- Load MIDI and timbre bank files into memory
- Provide interactive playback loop with keyboard controls (advance/rewind measures, seek, exit)
- Manage DOS text cursor visibility during playback
- Display real-time playback position in both time and measure:beat:tick formats

## External Dependencies
- **Local headers:** `music.h` (MUSIC_Init, MUSIC_PlaySong, MUSIC_SetSongPosition, MUSIC_GetSongPosition, MUSIC_GetSongLength, MUSIC_StopSong, MUSIC_Shutdown, MUSIC_ErrorString, MUSIC_SetVolume, MUSIC_RegisterTimbreBank); `sndcards.h` (GenMidi, SoundCanvas, Awe32, WaveBlaster, SoundBlaster, ProAudioSpectrum, SoundMan16, Adlib, SoundScape, UltraSound)
- **Standard C:** stdio, stdlib, string (printf, exit, malloc, free, strcpy, strcat, strlen, strnicmp, stricmp, fopen, fread, fclose, fseek, ftell, sscanf)
- **DOS/Platform-specific:** conio (kbhit, getch), dos (union REGS, int86, int386, SEEK_END, SEEK_SET)

# audiolib/public/pm/source/usrhooks.c
## File Purpose
Provides a modular memory management abstraction layer that wraps standard malloc/free operations. Designed as a "hook" module for library-level memory allocation that calling programs can modify to intercept or customize memory operations.

## Core Responsibilities
- Allocate memory with error checking and return standardized status codes
- Deallocate memory with null-pointer validation
- Provide abstraction point for custom memory management strategies
- Ensure dword-aligned pointer returns (per documentation)

## External Dependencies
- `stdlib.h` — provides `malloc()`, `free()`
- `usrhooks.h` — defines error enum (`USRHOOKS_Ok`, `USRHOOKS_Error`) and function prototypes

# audiolib/public/pm/source/usrhooks.h
## File Purpose
Public header file defining custom memory management hook functions for the audio library. Provides an abstraction layer allowing calling programs to override or intercept memory allocation and deallocation operations performed by the library.

## Core Responsibilities
- Define status/error codes for memory hook operations
- Declare memory allocation hook function prototype
- Declare memory deallocation hook function prototype
- Enable application-level control of library memory management

## External Dependencies
None. Self-contained header with no external includes or dependencies.

# audiolib/public/ps/ps.c
## File Purpose

A command-line sound player utility that initializes a sound card and plays audio files (WAV, VOC, or raw format). Users can configure sound card type, voice count, sample bits, sample rate, channels, and reverb via command-line arguments. The program loops on user input to replay the sound until ESC is pressed.

## Core Responsibilities

- Parse command-line arguments for sound card selection and audio configuration (voices, bits, channels, sample rate, reverb)
- Load audio files from disk with automatic file format detection (.wav, .voc, or raw)
- Initialize the FX sound manager with selected device and settings
- Play audio files using the appropriate format-specific playback function
- Handle synchronous user input and replay on keypress
- Clean up resources and shut down the audio system on exit

## External Dependencies

- **Includes:** `<conio.h>` (getch), `<dos.h>` (legacy DOS headers), `<stdlib.h>` (malloc, exit), `<stdio.h>` (fopen, fread, printf), `<string.h>` (strcpy, strcat, strlen, stricmp, strnicmp), `"fx_man.h"` (audio library API).
- **External symbols (defined elsewhere):** FX_SetupCard, FX_Init, FX_SetReverb, FX_SetVolume, FX_PlayWAV, FX_PlayVOC, FX_PlayRaw, FX_StopAllSounds, FX_Shutdown, FX_ErrorString, SoundBlaster, Awe32, ProAudioSpectrum, SoundMan16, SoundScape, UltraSound, SoundSource, TandySoundSource (sound card type constants from fx_man.h or sndcards.h).
- **Extern globals:** _argc, _argv (DOS runtime command-line storage).

# audiolib/public/ps/usrhooks.c
## File Purpose
Provides memory allocation and deallocation wrapper functions for the audio library. This module allows the calling program to intercept or customize memory operations required by the audio subsystem. The functions are intentionally left public for user modification.

## Core Responsibilities
- Wrap dynamic memory allocation via `malloc`
- Wrap dynamic memory deallocation via `free`
- Return standard error codes (`USRHOOKS_Ok`, `USRHOOKS_Error`) to indicate success/failure
- Ensure allocated memory satisfies dword-alignment requirement
- Validate pointers before deallocation

## External Dependencies
- `stdlib.h`: `malloc`, `free`
- `usrhooks.h`: Function declarations and error code enum

# audiolib/public/ps/usrhooks.h
## File Purpose
Public header for the USRHOOKS memory management hook system. Defines the interface contract that allows the audio library to delegate memory allocation and deallocation to the calling program, enabling custom memory management policies (pooling, tracking, restrictions, etc.).

## Core Responsibilities
- Define error codes for hook operations
- Declare memory allocation hook function signature
- Declare memory deallocation hook function signature
- Establish the interface that calling programs must implement to integrate with the audio library

## External Dependencies
- None; this is a pure interface definition with no external includes.

# audiolib/public/timer/source/timer.c
## File Purpose
A demonstration/test program showing how to use the TASK_MAN timer task scheduler. It creates multiple timers running at different rates, dynamically modifies their rates, and shows task termination. This is educational code from the Apogee Software era (1994–1995).

## Core Responsibilities
- Demonstrate task scheduler initialization and usage
- Create and schedule multiple independent timer tasks
- Show dynamic task rate modification via `TS_SetTaskRate`
- Demonstrate task termination via `TS_Terminate`
- Provide a simple timer callback function that increments counters
- Display running timer values in a loop

## External Dependencies
- **Standard headers:** `<stdio.h>` (printf), `<stdlib.h>`, `<conio.h>` (console I/O, DOS era), `<dos.h>` (DOS-specific)
- **Local header:** `"task_man.h"` (task struct definition and scheduler API)
- **Defined elsewhere:** `TS_ScheduleTask`, `TS_Dispatch`, `TS_SetTaskRate`, `TS_Terminate`, `TS_Shutdown` (task_man.c)

# audiolib/public/timer/source/usrhooks.c
## File Purpose
Provides wrapper functions for memory allocation and deallocation that serve as customization points for the audio library. The module allows the calling program to intercept or restrict memory operations while maintaining a consistent interface.

## Core Responsibilities
- Allocate dynamic memory with error checking
- Deallocate dynamic memory with validation
- Return standardized error codes (Ok/Error)
- Abstract malloc/free behind a library-controlled interface

## External Dependencies
- **Includes:** `<stdlib.h>` (malloc, free), `usrhooks.h` (local error code definitions)
- **Defined elsewhere:** All caller dependencies on `USRHOOKS_GetMem()` and `USRHOOKS_FreeMem()` throughout the audio library

# audiolib/public/timer/source/usrhooks.h
## File Purpose
Public header declaring memory management hook functions for the USRHOOKS module. Provides a callback interface that allows the calling program to intercept or customize memory allocation and deallocation operations required by the audio library.

## Core Responsibilities
- Define error codes for hook operations
- Declare allocation hook function (USRHOOKS_GetMem)
- Declare deallocation hook function (USRHOOKS_FreeMem)
- Establish a contract for custom memory management in the audio library

## External Dependencies
- Standard C (no explicit includes in this header)
- Caller must implement the two declared functions

