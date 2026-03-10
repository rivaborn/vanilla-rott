# audiolib/source/_al_midi.h
## File Purpose
Header file defining MIDI and AdLib FM synthesizer constants, data structures, and function declarations for the audio library. Establishes the interface for managing voices, channels, and instrument timbres on the AdLib sound card, a common DOS-era music synthesizer.

## Core Responsibilities
- Define MIDI message types, note values, and control constants
- Define hardware register constants for AdLib chip control
- Declare data structures for voice and channel state management
- Declare timbre (instrument) storage and retrieval
- Provide utility macros for byte manipulation and frequency calculation
- Declare voice and channel control functions

## External Dependencies
- **Defined elsewhere**: `ADLIB_TimbreBank` (defined in `_al_midi.c`)
- **Hardware constants**: `alFreqH` (0xb0), `alEffects` (0xbd) — AdLib register addresses
- **Macros**: `lobyte`, `hibyte` for 16-bit register pair splitting
- No C library dependencies visible; bare hardware abstraction

# audiolib/source/_blaster.h
## File Purpose
Private header defining Sound Blaster audio hardware constants, port addresses, DSP command codes, and mixer register definitions. Supports legacy SoundBlaster cards (1xx–4xx versions) for DOS-era audio playback and recording.

## Core Responsibilities
- Define I/O port offsets for Sound Blaster hardware communication (reset, read, write, data-available ports)
- Define mixer register addresses and control bits for audio configuration
- Define DSP (Digital Signal Processor) command opcodes for audio operations
- Provide helper macros for audio sampling rate and time-constant calculations
- Define format flags for audio data (signed/unsigned, mono/stereo)
- Parse BLASTER environment variable tokens (address, interrupt, DMA channels, card type)
- Define card capability tracking structure

## External Dependencies
- None (self-contained hardware definitions)

# audiolib/source/_guswave.h
## File Purpose

Private header file for GUS (Gravis Ultrasound) wave audio playback. Defines internal data structures and constants for managing voice channels, sound buffers, and hardware interaction in the audio subsystem.

## Core Responsibilities

- Defines playback state structures (VoiceNode, voice lists, status tracking)
- Declares WAV/VOC format parsing structures (RIFF, format, data chunks)
- Provides configuration constants (voice limits, buffer sizes, encoding types)
- Declares voice lifecycle functions (allocation, playback, format reading)
- Manages voice priority queuing and resource limits for concurrent playback

## External Dependencies

- No `#include` directives (private header)
- Uses `volatile` qualifier for hardware memory-mapped registers
- Function pointer callbacks (`GetSound`, `DemandFeed`) for format abstraction
- Direct hardware handle references (`GF1voice`) suggest Gravis Ultrasound driver integration

# audiolib/source/_midi.h
## File Purpose
Private C header file for MIDI song playback engine. Defines constants, message formats, and data structures for Standard MIDI file parsing and Extended MIDI (EMIDI) hardware-specific playback support. Part of the audio library's music playback subsystem authored by James R. Dose (Apogee Software, 1994–1995).

## Core Responsibilities
- Define MIDI protocol constants: message types, control codes, meta-event codes, hardware signatures
- Define playback state structures for tracks and song contexts with looping/timing metadata
- Declare private functions for MIDI event processing, timing, and hardware control
- Support EMIDI extensions for hardware-specific behavior (Adlib, Sound Canvas, etc.)
- Provide bit-manipulation and delta-time parsing helpers

## External Dependencies
- **Global state (defined elsewhere):** `_MIDI_Time`, `_MIDI_FPSecondsPerTick`, `_MIDI_Tick`, `_MIDI_Beat`, `_MIDI_Measure`, `_MIDI_Context`, `_MIDI_Funcs`, `_MIDI_PatchMap`
- **Type:** `task` (appears in service routine signature; likely OS/scheduler task type)
- **Implicit:** Sound driver interface via `_MIDI_Funcs->ProgramChange()`, `_MIDI_Funcs->ControlChange()` callbacks

# audiolib/source/_multivc.h
## File Purpose
Private header for the MULTIVOC audio mixing library. Defines structures, constants, and internal function declarations for multi-voice audio playback, mixing, and format support (VOC, WAV, raw). Implements audio mixing for different sample formats and stereo/mono configurations.

## Core Responsibilities
- Define voice node structure and doubly-linked list for active voices
- Declare audio format constants (8-bit, 16-bit, ADPCM, A-law, µ-law)
- Define WAV/VOC file format structures
- Declare internal mixing functions for different bit depths and channels
- Provide volume and pan lookup table infrastructure
- Declare voice lifecycle management (allocation, playback control, servicing)
- Define audio reverberation routines

## External Dependencies
- VGA graphics interface: `ATR_INDEX`, `STATUS_REGISTER_1`, `inp()`, `outp()` macros for border color modification (DOS-era hardware)
- No obvious external includes; assumes MULTIVOC.C defines the implementations
- Inline assembly (`#pragma aux`) assumes x86 instruction set

# audiolib/source/_pas16.h
## File Purpose
Private header for PAS16.C that defines constants, macros, and function declarations for ProAudio Spectrum 16 (PAS16) soundcard driver support. Provides hardware register addresses, bit flags, audio format definitions, and driver interface structures for retro-era audio hardware integration.

## Core Responsibilities
- Define hardware I/O port addresses and register layouts for PAS16 card
- Provide bit flag constants for audio modes (mono/stereo, 8-bit/16-bit) and hardware control
- Declare low-level register access functions (`PAS_Read`, `PAS_Write`)
- Declare card initialization and discovery functions (`PAS_FindCard`, `PAS_CheckForDriver`, `PAS_GetCardSettings`)
- Declare DMA and interrupt control functions (`PAS_SetupDMABuffer`, `PAS_EnableInterrupt`, `PAS_ServiceInterrupt`)
- Define structures for mapping hardware state (`MVState`) and function pointers (`MVFunc`)
- Provide macros for sample rate calculation and audio format encoding

## External Dependencies
- **No includes** (this is a private header; implementation in PAS16.C)
- **Assumes**: Watcom C/C++ compiler (uses `#pragma aux` for inline assembly and `far` keyword)
- **Real-mode x86 assumption**: Code targets 16-bit x86 DOS/real-mode environment (interrupt handlers, I/O port access, far pointers)
- **Symbols defined elsewhere**: All function implementations in PAS16.C; structures filled by driver calls

# audiolib/source/_sndscap.h
## File Purpose
Private header for low-level Ensoniq Soundscape sound card driver implementation. Defines hardware register offsets, firmware commands, audio format constants, and function declarations for direct hardware manipulation including interrupt handling, DMA control, and codec configuration.

## Core Responsibilities
- Ensoniq gate-array (ODIE/OPUS/MiMIC) hardware register offsets and indirect register addresses
- AD-1848 audio codec register definitions and indirect register layout
- Audio format mode flags (mono/stereo, 8/16-bit) and sample size macros
- Firmware command codes and status bit masks for device communication
- x86 interrupt controller definitions and IRQ handling setup
- Function declarations for gate-array I/O, interrupt management, DMA configuration, and device initialization

## External Dependencies
- **Standard C**: `FILE` type (for configuration parsing)
- **x86 Hardware**: Interrupt controller I/O ports (0x20, 0x21, 0xa0, 0xa1); DMA controller; port I/O via `__interrupt` pragma
- **DOS/DPMI Memory Model**: Protected-mode memory locking, far pointers, interrupt vector installation
- **External symbols**: Audio buffer management, hardware base address (not defined here)

# audiolib/source/adlibfx.c
## File Purpose
Low-level Adlib sound card driver for playing sound effects created by the Muse editor. Manages hardware initialization, sound playback control, and task-based audio updates. Handles a single monophonic Adlib voice with volume and priority management.

## Core Responsibilities
- Initialize and shut down the Adlib FX engine with DPMI memory locking
- Manage a single Adlib hardware voice for sound effect playback
- Send low-level register writes to the Adlib card (port 0x388)
- Schedule periodic service routine via task manager to advance playback
- Control sound volume (per-sound and global) with hardware register updates
- Implement priority-based voice stealing (don't play lower-priority sounds over current)
- Support completion callbacks when sounds finish playing

## External Dependencies

- **`dpmi.h`**: `DPMI_LockMemoryRegion()`, `DPMI_Lock()`, `DPMI_Unlock()`, `DPMI_UnlockMemoryRegion()` — manage real-mode memory protection in protected mode.
- **`task_man.h`**: `TS_ScheduleTask()`, `TS_Terminate()`, `TS_Dispatch()` — periodic task scheduling.
- **`interrup.h`**: `DisableInterrupts()`, `RestoreInterrupts()` — interrupt flag manipulation (inline asm).
- **`al_midi.h`**: Defines `ADLIB_PORT` (0x388); other Adlib card functions not used here.
- **`adlibfx.h`**: Header declaring public API and `ALSound` struct definition.
- **Standard C**: `<dos.h>`, `<stdlib.h>`, `<conio.h>` for DOS I/O (`outp()`, `inp()`); Watcom-specific.

# audiolib/source/adlibfx.h
## File Purpose
Public header for the ADLIBFX sound effects module. Defines the interface for Adlib sound card operations including voice management, playback control, volume management, and event callbacks for the legacy audio subsystem.

## Core Responsibilities
- Define error codes for Adlib operations (enum `ADLIBFX_Errors`)
- Define sound effect data structure (`ALSound`)
- Declare voice/sound lifecycle functions (Play, Stop, SoundPlaying)
- Declare voice availability and allocation APIs
- Declare volume control (per-sound and global)
- Declare callback registration for sound completion events
- Declare initialization and shutdown for the Adlib subsystem
- Declare memory locking primitives for DMA/hardware access

## External Dependencies
- No includes visible in this header.
- Implementation file (adlibfx.c) likely includes low-level hardware/DPMI APIs.
- `#pragma aux` directives use Watcom C calling convention extensions (frame attribute).

# audiolib/source/al_midi.c
## File Purpose
Low-level MIDI synthesizer driver for Adlib FM synthesis sound cards. Manages voice allocation, note playback, MIDI controller events, pitch tables, and hardware register communication. Supports mono/stereo and OPL2/OPL3 variants.

## Core Responsibilities
- FM voice lifecycle management (allocation, deallocation, reservation)
- MIDI event handling (note on/off, program change, control changes, pitch bend)
- Pitch table calculation and voice pitch programming
- Timbre (instrument) lookup and programming to hardware
- Volume and pan control per MIDI channel
- Hardware register I/O with timing constraints
- Adlib card detection and port configuration
- Interrupt-safe voice state management

## External Dependencies

- **Notable includes:** `<conio.h>` (I/O), `<dos.h>` (DOS), `<stdlib.h>` (standard library)
- **Local headers:** `dpmi.h` (memory locking), `interrup.h` (interrupt control), `sndcards.h` (card types), `blaster.h` (Sound Blaster config), `user.h` (user params), `al_midi.h`, `_al_midi.h` (MIDI types), `ll_man.h` (linked list)
- **Defined elsewhere:** `ADLIB_TimbreBank[]`, `ADLIB_PORT` constant, `outp()`/`inp()` system I/O, `LL_Remove()`, `LL_AddToTail()` linked-list ops, `hibyte()` macro, MIDI constants (`MIDI_VOLUME`, `MIDI_PAN`, etc.), voice/channel constants (`NUM_VOICES`, `NUM_CHANNELS`, `MAX_NOTE`, etc.), error codes (`AL_Ok`, `AL_Error`, `AL_VoiceNotFound`), `DPMI_*()` functions, `BLASTER_*()` functions, `DisableInterrupts()`, `RestoreInterrupts()`, `USER_CheckParameter()`.

# audiolib/source/al_midi.h
## File Purpose
Header file for the audio library's MIDI/FM synthesis (Adlib) interface. Defines error codes, hardware constants, and function declarations for voice management, MIDI note/control operations, and hardware initialization on legacy FM synthesizer cards.

## Core Responsibilities
- Define error codes and MIDI/audio constants (volume ranges, pitch bend, hardware port)
- Declare voice allocation and release functions
- Declare MIDI event handlers (note on/off, program change, control change)
- Declare hardware initialization, detection, and shutdown
- Declare stereo configuration and timbre bank loading
- Declare low-level register I/O to hardware

## External Dependencies
- No includes (header-only declarations)
- Assumes caller provides MIDI channel/voice numbers and hardware port mappings
- Hardware: Adlib/FM synthesizer at port `0x388`

# audiolib/source/assert.h
## File Purpose
Debug assertion macro library for runtime condition checking. Provides a conditional assertion facility that calls an error reporting function on failure in debug builds and compiles to nothing in release builds (NDEBUG mode).

## Core Responsibilities
- Define `ASSERT` macro for condition checking in debug mode
- Declare external assertion failure handler (`_Assert`)
- Support Watcom C++ compiler conventions via pragma directives
- Provide guard against multiple header inclusions

## External Dependencies
- `__FILE__`, `__LINE__` preprocessor constants (C standard)
- `#pragma aux` directive (Watcom C/C++ compiler-specific)
- External symbol: `_Assert` (defined elsewhere in audiolib)

# audiolib/source/awe32.c
## File Purpose
Provides wrapper functions for the AWE32 sound card driver on DOS systems, enabling MIDI playback and hardware control. Serves as a translation layer between the game engine and the AWE32 low-level library, handling initialization, shutdown, and real-time MIDI operations.

## Core Responsibilities
- Detect and initialize AWE32 sound card hardware and MIDI interface
- Provide MIDI control functions (note on/off, pitch bend, program change, aftertouch)
- Manage base I/O addresses for Sound Blaster, EMU8000, and MPU-401
- Lock time-critical code and data into memory for real-time safety via DPMI
- Track active notes per channel for cleanup operations
- Load built-in SoundFont presets and configure banks
- Report errors via centralized error code system

## External Dependencies
- **Includes**: `conio.h` (I/O), `string.h` (memset), `dpmi.h` (memory locking), `blaster.h` (Sound Blaster), `ctaweapi.h` (AWE32 API)
- **External symbols**: `awe32NoteOn`, `awe32NoteOff`, `awe32ProgramChange`, `awe32Controller`, `awe32PolyKeyPressure`, `awe32ChannelPressure`, `awe32PitchBend`, `awe32Detect`, `awe32InitHardware`, `awe32InitMIDI`, `awe32Terminate`, `awe32InitNRPN`, `awe32TotalPatchRam`, `awe32DefineBankSizes`, `awe32SoundPad`, `awe32SPadXObj` (1–7), `awe32NumG`, `__midieng_code`, `__midieng_ecode`, `__nrpn_code`, `__nrpn_ecode`, `__midivar_data`, `__nrpnvar_data`, `__embed_data`, `BLASTER_GetCardSettings`, `BLASTER_GetEnv`, `BLASTER_ErrorString`, `DPMI_LockMemoryRegion`, `DPMI_Lock`, `DPMI_Unlock`, `DPMI_UnlockMemoryRegion`, `DPMI_UnlockMemory`, `DPMI_LockMemory`, `inp`, `outp`

# audiolib/source/awe32.h
## File Purpose
Public header file that defines the interface for the AWE32 Sound Blaster synthesizer driver. Provides error codes and MIDI control functions for initializing and operating the AWE32 audio chip, which was a common Sound Blaster soundcard used in 1990s PC games for MIDI music playback and sound synthesis.

## Core Responsibilities
- Define error codes for AWE32 operations
- Declare device initialization and shutdown
- Declare MIDI note on/off and note control functions
- Declare MIDI channel control (aftertouch, program change, control change, pitch bend)
- Provide error-to-string conversion

## External Dependencies
None (header-only declarations). Implementation in awe32.c would directly interface with AWE32 hardware or DOS/Windows DPMI and MPU-401 MIDI interfaces.

# audiolib/source/blaster.c
## File Purpose

Low-level Sound Blaster driver for DOS, supporting multiple card versions (1.xx, Pro, 2.xx, and 16). Handles DSP communication, DMA-based audio playback/recording, mixer control, and interrupt servicing for digitized sound output.

## Core Responsibilities

- DSP (Digital Signal Processor) communication via port I/O (read, write, reset, version detection)
- DMA buffer setup and management; tracks current playback position across circular buffer
- Hardware interrupt handling; chains to old handler if interrupt not from Sound Blaster
- Playback/record initiation tailored to DSP version (different command sequences for 1.xx, 2.xx, 4.xx)
- Audio format and sample-rate configuration (mono/stereo, 8-bit/16-bit, rate bounds per card)
- Mixer control: volume adjustment for voice/MIDI channels; stereo mode switching
- DPMI memory locking for interrupt-safe code and buffer regions
- Configuration parsing from BLASTER environment variable; card capability detection
- Error reporting and status tracking

## External Dependencies

- **DPMI** (dpmi.h): DPMI_LockMemory, DPMI_UnlockMemory, DPMI_UnlockMemoryRegion, DPMI_LockMemoryRegion for memory protection.
- **DMA** (dma.h): DMA_SetupTransfer, DMA_EndTransfer, DMA_GetCurrentPos, DMA_VerifyChannel, DMA_ErrorString for DMA controller setup.
- **IRQ** (irq.h): IRQ_SetVector, IRQ_RestoreVector for high-IRQ (8–15) interrupt installation.
- **DOS/Watcom built-ins**: inp, outp (port I/O), int386, _dos_getvect, _dos_setvect, _chain_intr (interrupt vectors).
- **Standard C**: getenv, sscanf, toupper, isxdigit, memset, min, max.

# audiolib/source/blaster.h
## File Purpose
Public header for the BLASTER audio library, providing a C interface to Sound Blaster compatible audio cards. Defines configuration structures, error codes, card types, and function declarations for audio playback, recording, DSP control, and mixer operations on legacy Sound Blaster hardware.

## Core Responsibilities
- Define Sound Blaster hardware configuration and capability structures
- Declare error codes, card types, and audio format constants
- Provide DSP (Digital Signal Processor) read/write and reset operations
- Manage audio playback and recording with buffering and DMA
- Control mixer hardware (volume, speaker on/off)
- Handle interrupt and DMA channel configuration
- Provide callback mechanisms for audio completion events
- Lock/unlock memory for DMA-safe operations

## External Dependencies
- None visible in this header (standard C types only)
- Implementation (blaster.c) likely includes: ISA hardware port I/O, DOS/DPMI memory locking, interrupt handling

# audiolib/source/blastold.c
## File Purpose

Low-level driver for Sound Blaster sound cards (SB 1.0 through SB16) in protected mode (DOS). Manages DSP communication, DMA transfers, interrupt handling, and mixer control to enable digital audio playback and recording.

## Core Responsibilities

- Initialize and shut down Sound Blaster hardware via DSP command sequences
- Handle hardware interrupts on DMA completion, manage circular audio buffers, and invoke user callbacks
- Parse BLASTER environment variable and validate card configuration
- Set up and manage DMA channels for audio data transfer (8-bit and 16-bit)
- Support multiple card types with version-specific DSP command sets (1.xx, 2.xx, 4.xx)
- Control mixer chip for volume and audio source management
- Lock/unlock kernel memory and allocate protected-mode stack for interrupt handler
- Provide error reporting and environment variable handling

## External Dependencies

- **dos.h, conio.h:** Low-level port I/O (`inp`, `outp`), DOS interrupt vectors (`_dos_getvect`, `_dos_setvect`), register structures (`union REGS`, `int386`)
- **stdlib.h, stdio.h, string.h, ctype.h:** Standard C utilities
- **dpmi.h:** Protected-mode memory locking and DPMI calls (defined elsewhere; uses `DPMI_Lock`, `DPMI_Unlock`, `DPMI_LockMemoryRegion`, `DPMI_UnlockMemoryRegion`)
- **dma.h:** DMA controller setup (defined elsewhere; uses `DMA_SetupTransfer`, `DMA_EndTransfer`, `DMA_GetCurrentPos`, `DMA_VerifyChannel`)
- **irq.h:** Higher IRQ vector support (defined elsewhere; uses `IRQ_SetVector`, `IRQ_RestoreVector`)
- **blaster.h:** Public interface and `BLASTER_CONFIG` struct
- **_blaster.h:** Private constants and `CARD_CAPABILITY` struct
- **Inline assembly pragmas:** `GetStack`, `SetStack` for stack manipulation in interrupt handler

**Not inferable from this file:** Actual implementation of DMA, DPMI, IRQ, and timer interrupt handling; depends on platform-specific modules.

# audiolib/source/ctaweapi.h
## File Purpose
Header file defining the Sound Blaster AWE32 audio interface API for DOS. Declares hardware control functions, MIDI/NRPN support, SoundFont management, and device state variables. Compiled for multiple C compilers with platform-specific memory and calling conventions.

## Core Responsibilities
- Define cross-compiler type aliases and macros (BYTE, WORD, DWORD, FAR pointers)
- Declare hardware register access functions (awe32RegW, awe32RegRW, etc.)
- Export MIDI support functions (note on/off, program change, controllers, pitch bend)
- Export NRPN (Non-Registered Parameter Number) initialization
- Declare SoundFont and wave packet streaming functions for sample data loading
- Define data structures (SOUND_PACKET, WAVE_PACKET) for metadata during streaming
- Manage struct packing and name mangling directives for __WATCOMC__, __HIGHC__, __SC__ compilers

## External Dependencies
- **Compiler directives**: `__FLAT__`, `__HIGHC__`, `DOS386`, `__SC__`, `__WATCOMC__` for struct packing and calling conventions
- **All functions and variables declared `extern`**: Implementations in linked modules (`__midieng_code`, `__hardware_code`, `__sbkload_code`, `__nrpn_code`)
- **No standard library includes** (pure hardware API)

# audiolib/source/debugio.c
## File Purpose
Legacy DOS-era debug output library that writes directly to video memory (monochrome adapter at 0xb0000). Provides character, string, and formatted printing to an on-screen debug console with cursor positioning and auto-scrolling.

## Core Responsibilities
- Direct memory I/O to VGA/MDA display buffer (0xb0000)
- Cursor positioning and line management
- Screen scrolling when display buffer is full
- Character output with printable ASCII filtering
- Number-to-string conversion (decimal, hex, unsigned)
- Printf-style variadic formatted output with %d, %s, %u, %x specifiers

## External Dependencies
- **Includes**: `<stdio.h>` (EOF), `<stdarg.h>` (va_list, va_start, va_end), `<stdlib.h>` (included but unused), `"debugio.h"` (public API declarations)
- **Direct memory access**: Hardcoded to VGA monochrome display buffer at `0xb0000`
- **External symbols**: None; all helper functions defined locally (static)

# audiolib/source/debugio.h
## File Purpose
Debug I/O interface header providing character and string output functions. Declares a small set of utilities for formatted debug output to an unspecified device or buffer, likely used during development and runtime debugging of the audio library.

## Core Responsibilities
- Set cursor position for debug output (DB_SetXY)
- Write individual characters to debug output (DB_PutChar)
- Output strings and numbers in various formats (DB_PrintString, DB_PrintNum, DB_PrintUnsigned)
- Provide printf-style formatted output interface (DB_printf)

## External Dependencies
- Standard C variadic macro system (for DB_printf)
- Implementation defined elsewhere (debugio.c or platform-specific code)

# audiolib/source/dma.c
## File Purpose
Low-level DMA controller driver for ISA bus 8-bit and 16-bit transfers. Provides hardware abstraction for configuring Intel 8237 DMA controller channels via I/O port programming, with error reporting and transfer status queries. Targets DOS/early Windows ISA architecture.

## Core Responsibilities
- Channel validation and error reporting for DMA operations
- Configure DMA controller with address, length, and transfer mode (single-shot/auto-init, read/write)
- Enable/disable DMA channels via hardware mask ports
- Query live transfer position and remaining byte count during active DMA
- Abstract 8-bit (channels 0–3) and 16-bit (channels 5–7) DMA port mappings

## External Dependencies
- **`<dos.h>`** – `outp()`, `inp()` for 8086 ISA I/O port read/write.
- **`<conio.h>`** – Console/hardware I/O (may overlap with dos.h).
- **`<stdlib.h>`** – Standard C library.
- **`dma.h`** – Local header defining error and mode enums and public function signatures.

---

**Notes:**
- Author: James R. Dose (1994).
- Hard-coded 8237 DMA controller port addresses assume ISA bus (not PCI/modern).
- Hardware-specific: `0xA`, `0xB`, `0xC` for 8-bit; `0xD4`, `0xD6`, `0xD8` for 16-bit.
- Uses bitwise operations and shifts heavily; address reconstruction is non-obvious.

# audiolib/source/dma.h
## File Purpose
Public header for DMA (Direct Memory Access) operations, providing an abstraction layer for setting up and monitoring memory-to-device transfers. Part of the audio library, likely used for real-time audio data streaming to sound hardware.

## Core Responsibilities
- Define error codes and transfer mode constants for DMA operations
- Declare channel verification and setup functions
- Declare transfer control and monitoring functions
- Provide error-to-string conversion for diagnostics

## External Dependencies
None—self-contained header with no includes or external symbol references.

# audiolib/source/dpmi.c
## File Purpose

Implements DOS Protected Mode Interface (DPMI) functionality for managing real-mode interrupts and memory locking in a DOS/4GW protected-mode environment. Provides abstraction over DPMI 0x31 interrupt calls to get/set real-mode interrupt vectors, invoke real-mode functions, and lock/unlock memory regions for DMA-safe operations.

## Core Responsibilities

- Get and set real-mode interrupt vectors (DPMI functions 0x0200/0x0201)
- Execute real-mode procedures with register state preservation (DPMI function 0x0301)
- Lock memory regions to prevent paging (DPMI function 0x0600)
- Unlock previously-locked memory regions (DPMI function 0x0601)
- Manage CPU and segment register state for DPMI calls
- Convert pointer addresses to linear addresses for memory locking operations

## External Dependencies

- **`<dos.h>`** – `union REGS`, `struct SREGS`, `int386()`, `int386x()`, `FP_SEG()`, `FP_OFF()` macros for register and interrupt manipulation
- **`<string.h>`** – included but not used in this file
- **`dpmi.h`** – local header defining `dpmi_regs` struct, `DPMI_Errors` enum, and function prototypes
- **DPMI BIOS interrupt 0x31** – invoked via `int386()` and `int386x()` for DPMI services

# audiolib/source/dpmi.h
## File Purpose
Header providing DPMI (DOS Protected Mode Interface) wrappers for low-level x86 DOS operations. Enables protected-mode code to interact with real-mode DOS functionality, including memory management, interrupt vectors, and real-mode function calls. Designed for DOS extender environments (e.g., DOS/4GW).

## Core Responsibilities
- Define DPMI error codes and register state structure
- Declare functions for DOS memory allocation and deallocation
- Provide memory locking/unlocking utilities for DMA-safe regions
- Offer real-mode interrupt vector manipulation
- Enable calling real-mode DOS functions from protected mode
- Implement low-level operations via x86 inline assembly (pragma aux)

## External Dependencies
- **x86 CPU:** int 31h (DPMI interrupt).
- **DOS Extender (e.g., DOS/4GW):** Provides DPMI services.
- **Watcom C compiler:** pragma aux syntax for inline assembly.
- No external headers included.

# audiolib/source/fx_man.c
## File Purpose
Device-independent sound effects manager providing a high-level API for audio playback, recording, and device abstraction. Acts as a facade layer over multiple sound card drivers (Sound Blaster, PAS16, SoundScape, Ultrasound, SoundSource), delegating most functionality to the MULTIVOC mixer library.

## Core Responsibilities
- Sound device initialization, configuration, and shutdown across multiple hardware types
- Sound playback in multiple formats (VOC, WAV, raw) with looping and 3D positioning
- Voice management (voice availability checking, playback control)
- Audio property manipulation (volume, panning, pitch, frequency, reverb)
- Recording initiation and termination
- Error code mapping to user-readable messages
- Device-specific error handling and delegation to appropriate driver

## External Dependencies
- **Standard Library:** stdio.h, stdlib.h
- **Sound card drivers:** blaster.h, pas16.h, sndscape.h, guswave.h, sndsrc.h
- **Core mixer:** multivoc.h (MULTIVOC library – does actual voice mixing and playback)
- **Device enumeration:** sndcards.h (sound card type constants)
- **Memory management:** ll_man.h (low-level DMA/IRQ memory locking)
- **User input:** user.h (USER_CheckParameter for command-line/env var checks)
- **Self:** fx_man.h (this module's public interface)

**Defined elsewhere:** All device-specific drivers (BLASTER, PAS, SOUNDSCAPE, GUSWAVE, SS), MULTIVOC mixer, memory locker (LL_*), user parameter parser

# audiolib/source/fx_man.h
## File Purpose
Public header for the sound effects manager (FX_MAN.C) in a 1994 Apogee Software game engine. Provides the primary API for sound card initialization, audio playback, effects processing, spatial audio, and recording functionality.

## Core Responsibilities
- Sound card detection, configuration, and initialization (Sound Blaster support)
- Sound file playback (VOC, WAV, raw formats) with looping and priority management
- Voice management (allocation based on priority, voice availability checks)
- Audio effects (reverb, pitch shifting, frequency control, panning, 3D spatial audio)
- Master volume and audio parameter control
- Sound lifecycle management (stop, callback-based completion notification)
- Demand-feed playback for streaming audio
- Audio recording capture with callback-based sample delivery
- Error handling and device capability reporting

## External Dependencies
- **Include**: `sndcards.h` — enumeration of supported sound card types.
- **Defined Elsewhere**: Sound card driver implementations, ISR/DMA handlers, mixer algorithm (all in fx_man.c and subordinate modules).

# audiolib/source/gmtemp.c
## File Purpose
Defines a bank of pre-configured AdLib OPL synthesizer patches (timbres) for FM synthesis. Contains 256 hardcoded TIMBRE entries that represent different instrument sounds and effects for the AdLib sound card.

## Core Responsibilities
- Provide a global timbre bank with 256 pre-initialized synthesizer patches
- Define the TIMBRE data structure for AdLib FM synthesis parameters
- Enable sound/music modules to access standardized instrument definitions
- Supply patch parameters for two operator FM synthesis (modulator and carrier)

## External Dependencies
- Standard C library includes (implicit via GPL license header)
- Likely consumed by AdLib driver or music/sound system files that reference `ADLIB_TimbreBank`

---

**Notes:**
- TIMBRE structure maps directly to AdLib/OPL hardware registers: `SAVEK`, `Level`, `Env1`, `Env2`, `Wave` correspond to operator parameters; `Feedback` controls algorithm/feedback; `Transpose` adjusts pitch
- Array entries 128–255 contain many repeated or blank entries (particularly indices 128–160 repeat the same patch, and indices 161+ have varied transpose values), suggesting incomplete or placeholder patch definitions
- This is typical of legacy DOS game audio—pre-computed patch tables avoid runtime synthesis overhead

# audiolib/source/gmtimbre.c
## File Purpose
Provides a static timbre (instrument) bank for the AdLib sound card, containing 256 FM synthesizer parameter configurations for audio synthesis. This is a direct data table mapping MIDI instruments to hardware synthesizer register settings.

## Core Responsibilities
- Defines the TIMBRE struct encoding FM operator parameters (envelope, level, waveform)
- Supplies a hardcoded lookup table of 256 instrument definitions for AdLib compatibility
- Encodes MIDI-related metadata (transpose, velocity sensitivity) per instrument

## External Dependencies
None. This file contains only struct definition and data initialization.


# audiolib/source/gmtmbold.c
## File Purpose
Defines AdLib FM synthesis timbre data and MIDI percussion-to-timbre mappings for a game audio system. Contains pre-configured instrument parameters and drum note lookup tables used during MIDI playback.

## Core Responsibilities
- Provide 174 pre-configured FM synthesis timbres/instruments for AdLib sound generation
- Map MIDI percussion note numbers (0–127) to appropriate drum timbres and key values
- Supply FM operator parameters (envelope, waveform, feedback, sustain levels) for both operators
- Support dynamic instrument selection during MIDI playback

## External Dependencies
- No includes or external symbols — self-contained data module
- Implicitly used by other audio subsystem code that performs MIDI→AdLib synthesis conversion


# audiolib/source/gus.c
## File Purpose
Provides initialization and shutdown routines for the Gravis Ultrasound (GUS) sound card hardware. Manages GUS OS loading, conventional memory allocation for DMA buffers, and error reporting for both MIDI and digital audio playback.

## Core Responsibilities
- Initialize and shut down GUS hardware with reference counting
- Allocate DOS conventional memory for DMA operations via DPMI
- Map error codes to human-readable error messages
- Integrate GUS-specific initialization into broader audio library (gusmidi, guswave)
- Query and cache available GUS DRAM configuration

## External Dependencies
- **Interrupt/Memory:** `int386()`, `union REGS` from `dos.h` — DPMI calls for DOS memory allocation
- **GF1 Hardware Layer:** `gf1_load_os()`, `gf1_unload_os()`, `gf1_mem_avail()`, `gf1_error_str()`, `gf1_free()` from `newgf1.h`
- **Configuration:** `GetUltraCfg()` — loads ULTRAMID.INI; location defined elsewhere
- **Wave Playback:** `GUSWAVE_Voices[]`, `GUSWAVE_Installed`, `GUSWAVE_KillAllVoices()` from `guswave.h` / `_guswave.h`
- **Standard Library:** `strerror()` from `string.h`

# audiolib/source/gusmidi.c
## File Purpose
Implements MIDI music playback for the Gravis Ultrasound sound card. Handles loading instrument patches from disk (.pat files), managing MIDI message routing (program changes, note on/off, pitch bend, control changes), and providing initialization/shutdown routines for the GUS hardware.

## Core Responsibilities
- Load and unload instrument patches from disk into GUS DRAM, with interrupt-safe memory management
- Parse MIDI configuration files (ULTRAMID.INI) to map MIDI program numbers to hardware patch indices
- Route MIDI messages (program change, note on/off, control change, pitch bend) to the GF1 driver
- Manage master volume control for MIDI synthesis
- Maintain patch metadata: filenames, load status, waveform memory pointers across 256 melodic and percussion instruments
- Initialize and shutdown GUS hardware with configuration validation

## External Dependencies
**Standard headers**: `<conio.h>`, `<dos.h>`, `<stdio.h>`, `<io.h>`, `<fcntl.h>`, `<string.h>`, `<stdlib.h>` (DOS/Turbo C era)  
**Local headers**:
- `"usrhooks.h"` — Memory allocation hooks (`USRHOOKS_GetMem`, `USRHOOKS_FreeMem`)
- `"interrup.h"` — Interrupt control macros (`DisableInterrupts`, `RestoreInterrupts`)
- `"newgf1.h"` — GUS driver API (`gf1_*` functions, structures)
- `"gusmidi.h"` — Module public interface and error codes

**External symbols (defined elsewhere)**:
- `GUS_HoldBuffer`, `GUS_TotalMemory`, `GUS_MemConfig`, `GUS_ErrorCode`, `GUS_AuxError` — GUS driver state
- `GUS_Init()`, `GUS_Shutdown()` — Hardware initialization
- All `gf1_*()` functions — Gravis Ultrasound driver API (patch I/O, MIDI synthesis, DMA)

# audiolib/source/gusmidi.h
## File Purpose
Header file defining the public interface for Gravis UltraSound (GUS) MIDI control. Declares error codes, MIDI note/control functions, patch management, and initialization routines for sound card hardware abstraction.

## Core Responsibilities
- Define GUS error/status codes enum for return values
- Declare MIDI event functions (note on/off, pitch bend, control change, program change)
- Declare patch loading/unloading and mapping functions
- Declare volume control functions
- Declare GUS hardware initialization and shutdown
- Provide error string conversion utility

## External Dependencies
- `struct gf1_dma_buff` (GF1 refers to GUS chipset; struct defined elsewhere)
- Watcom C compiler pragmas (`#pragma aux`) for low-level hardware calling conventions
- DOS memory allocation patterns (`D32DosMemAlloc`)
- Apogee Software copyright (1994-1995)

**Notes:** This is legacy DOS-era code targeting the Gravis UltraSound card via Watcom C. The pragmas and DOS memory references indicate real-mode or protected-mode DOS compilation.

# audiolib/source/gusmidi2.c
## File Purpose

MIDI music driver for the Gravis Ultrasound audio card. Manages patch (instrument) loading/unloading, MIDI event dispatch, volume control, and hardware initialization. Enables the game engine to play General MIDI music via GUS synthesis hardware.

## Core Responsibilities

- Load and unload instrument patches from disk into GUS memory
- Parse ULTRAMID.INI configuration to map MIDI program numbers to patch files
- Dispatch MIDI events (note on/off, program change, control change, pitch bend) to GF1 driver
- Allocate and manage GUS DRAM across multiple hardware configurations (256K–1MB)
- Initialize/shutdown GUS hardware with reference counting
- Report errors through error code and human-readable message strings
- Control master volume (0–255 range)

## External Dependencies

**Standard C:**
- `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<malloc.h>`, `<math.h>`, `<limits.h>`, `<io.h>`, `<fcntl.h>`

**DOS/platform-specific:**
- `<conio.h>`, `<dos.h>` — console I/O and DOS interrupts (for DPMI via `int386()`).

**Local headers:**
- `gusmidi.h` — public API and error codes.
- `newgf1.h` — GUS hardware abstraction and patch structures.

**External symbols (defined elsewhere):**
- `gf1_load_os()`, `gf1_unload_os()` — GUS driver load/unload.
- `gf1_get_patch_info()`, `gf1_load_patch()`, `gf1_unload_patch()` — patch management.
- `gf1_mem_avail()` — query available GUS memory.
- `gf1_midi_*()` — MIDI event dispatch (note on/off, program change, volume, pitch bend, etc.).
- `gf1_error_str()` — GF1 error message string.
- `GetUltraCfg()` — read hardware configuration from ULTRASND.INI.

# audiolib/source/gusmidi2.h
## File Purpose
Public interface header for Gravis Ultrasound (GUS) MIDI synthesizer support. Declares functions for initializing/shutting down the GUS system, loading instrument patches, and sending MIDI events (note on/off, control change, pitch bend, program change).

## Core Responsibilities
- Define GUS error codes and translate to human-readable strings
- Initialize and shut down GUS hardware and MIDI subsystems
- Load and unload instrument patches into GUS DRAM
- Send MIDI control messages (note on/off, pitch bend, program change, control change)
- Manage master volume for the GUS output
- Allocate DOS-accessible memory for GUS operations

## External Dependencies
- Standard C library (implied: `char *`, `int` pointer operations)
- Not inferable: DOS extender headers, GUS hardware interface headers, patch file format definitions

# audiolib/source/guswave.c
## File Purpose

Digitized sound playback driver for the Gravis Ultrasound (GUS) audio card. Manages voice allocation, audio format parsing (VOC/WAV), real-time pitch/volume/pan control, and interrupt-driven playback callbacks on DOS-era hardware.

## Core Responsibilities

- **Voice management**: allocate/deallocate voice slots from a fixed pool, with priority-based preemption
- **Audio format support**: parse VOC and WAV file formats; support demand-fed streaming
- **Playback control**: start/stop playback, manage sample rate and bit depth, coordinate with GUS hardware
- **Pitch scaling**: apply pitch offset to playback rate via lookup table
- **3D panning**: map angle/distance to pan and volume for positional audio
- **Volume control**: master volume and per-voice volume with scaling
- **Interrupt handling**: provide callback handlers for GUS completion interrupts; maintain interrupt-safe state
- **Error reporting**: map error codes to human-readable messages

## External Dependencies

**Standard C / DOS headers**:
- `<stdlib.h>`, `<conio.h>`, `<dos.h>`, `<stdio.h>`, `<io.h>`, `<string.h>` – memory, I/O, DOS services

**Custom headers**:
- `debugio.h` – debug output (`DB_printf()`, `DB_PrintNum()`)
- `interrup.h` – interrupt control (`DisableInterrupts()`, `RestoreInterrupts()`)
- `ll_man.h` – linked list macros (`LL_AddToTail()`, `LL_Remove()`)
- `pitch.h` – pitch scaling (`PITCH_GetScale()`)
- `user.h` – command-line parameter checking (`USER_CheckParameter()`)
- `multivoc.h` – audio library API (error codes like `MV_InvalidVOCFile`)
- `_guswave.h` – private guswave declarations (likely `VoiceNode`, `voicestatus`, etc.)
- `newgf1.h` – GUS hardware interface (`gf1_play_digital()`, `gf1_stop_digital()`, etc.)
- `gusmidi.h` – MIDI subsystem detection (`GUSMIDI_Installed`)
- `guswave.h` – public header for this module

**External symbols (defined elsewhere)**:
- `GUS_Init()`, `GUS_Shutdown()`, `GUS_Error`, `GUS_ErrorString()` – GUS hardware initialization/control
- `gf1_play_digital()` – start hardware voice playback with callback
- `gf1_stop_digital()` – stop hardware voice
- `gf1_dig_set_freq()`, `gf1_dig_set_pan()`, `gf1_dig_set_vol()` – hardware voice parameter updates
- `gf1_malloc()` – allocate GUS onboard memory
- `GUS_HoldBuffer` – internal GUS buffer for playback handoff
- `GF1BSIZE` – GUS voice buffer size constant
- `GUSMIDI_Installed` – MIDI module presence flag
- `MV_ErrorCode` – multivoc error reporting
- `GUSWAVE_MinVoiceHandle`, `MAX_VOICES`, `VOICES`, `MAX_VOLUME`, `VOC_8BIT`, `VOC_16BIT`, `MAX_BLOCK_LENGTH` – constants (likely from headers)

# audiolib/source/guswave.h
## File Purpose
Public API header for the GUS (Gravis UltraSound) Wave audio module. Defines the interface for playing sampled audio (WAV/VOC) files through GUS hardware, supporting voice management, 3D audio positioning, and real-time playback control.

## Core Responsibilities
- Voice allocation and lifecycle management (init, play, kill, shutdown)
- WAV and VOC file playback with pitch/pan control
- Demand-feed (callback-driven) playback for dynamic audio generation
- Volume control and stereo reversal configuration
- Error reporting for GUS hardware and audio operations
- Voice activity tracking and availability queries

## External Dependencies
- `stdio.h` (implied by char pointer and error string API)
- GUS hardware driver layer (not inferable; implemented in guswave.c)
- Watcom C compiler (pragma aux directives)

# audiolib/source/interrup.h
## File Purpose
Provides low-level CPU interrupt flag manipulation through inline x86 assembly. Used to disable interrupts before critical sections and restore them afterward, enabling atomic operations in the audio library.

## Core Responsibilities
- Disable CPU interrupts and return current CPU state (EFLAGS)
- Restore CPU interrupts to a previously saved state
- Support interrupt-safe critical sections in the audio system

## External Dependencies
- **Compiler-specific**: Watcom C `#pragma aux` directives for inline assembly; not portable to other C compilers.
- No external symbols referenced.

# audiolib/source/irq.c
## File Purpose
Low-level DOS Protected Mode Interface (DPMI) interrupt handler management. Bridges between real mode and protected mode interrupt handlers, allowing safe interrupt vector setup and restoration in a DOS extender environment (1994–1995 Apogee era).

## Core Responsibilities
- Allocate DOS conventional memory for real mode interrupt stubs via DPMI function 0x0100
- Set up DPMI real mode callbacks (function 0x0303) to translate real mode → protected mode calls
- Install protected and real mode interrupt vectors via DPMI functions 0x0205 and 0x0201
- Restore original interrupt handlers on cleanup
- Manage callback registration/deregistration through DPMI functions 0x0304

## External Dependencies
- **Includes:** `<dos.h>` (int386, int386x, REGS, SREGS, FP_SEG, FP_OFF, MK_FP), `<stdlib.h>` (unused), `irq.h` (local header with error enum and function declarations)
- **DPMI services:** Interrupt 0x31 (DPMI) with functions 0x0100 (allocate DOS memory), 0x0200/0x0201 (get/set RM vector), 0x0204/0x0205 (get/set PM vector), 0x0303 (allocate callback), 0x0304 (free callback)
- **Macros from dos.h:** `FP_SEG()`, `FP_OFF()`, `MK_FP()` for far pointer manipulation

# audiolib/source/irq.h
## File Purpose
Public header declaring interrupt request (IRQ) management functions for the audio library. Provides the interface for installing and restoring interrupt vector handlers, used during audio system initialization and shutdown to manage hardware interrupts (likely for sound cards).

## Core Responsibilities
- Define error codes for IRQ operations
- Provide validation macro for IRQ numbers
- Declare interrupt vector installation function
- Declare interrupt vector restoration function

## External Dependencies
- Uses `__interrupt` calling convention keyword (compiler-specific, likely Watcom C or similar DOS-era compiler)
- Targets low-level x86 interrupt architecture (DOS/real-mode environment based on copyright era and IRQ range 0–15)

# audiolib/source/leeold.c
## File Purpose
Pure data file defining AdLib synthesizer instrument and percussion configurations. Provides 256 instrument timbres and 128 percussion note mappings for FM synthesis-based music and sound effects in the game engine.

## Core Responsibilities
- Store FM synthesis parameters (SAVEK, envelope, waveform, feedback) for 256 instruments
- Define percussion MIDI note-to-timbre mapping for drum sounds
- Serve as lookup tables for sound engine initialization and playback
- Encode instrument presets compatible with AdLib sound card hardware

## External Dependencies
- Standard C types only (`unsigned char`).
- No includes or external function calls.

# audiolib/source/leetimb1.c
## File Purpose
Data definition file containing timbre (FM synthesis instrument) configurations and percussion mappings for the AdLib FM audio engine. Provides lookup tables for 256 instrument timbres and 128 MIDI percussion drum sounds used during audio synthesis and music playback.

## Core Responsibilities
- Define FM synthesis timbre parameters (envelope, waveform, feedback settings)
- Provide percussion-to-timbre mapping for MIDI drum notes (channels 120–127)
- Store complete AdLib timbre bank with factory presets for game instruments
- Support initialization of audio synthesis engine with instrument definitions

## External Dependencies
- **Standard C types only** (unsigned char, implicit array sizing)
- **Defined elsewhere**: Functions that read these tables (audio driver, MIDI playback engine) not visible in this file
- **No external includes**

# audiolib/source/leetimbr.c
## File Purpose
Defines a lookup table of AdLib FM synthesizer instrument timbres/patches. The `ADLIB_TimbreBank` array contains 256 pre-configured instrument definitions that map to standard AdLib instrument slots used during gameplay audio synthesis.

## Core Responsibilities
- Store immutable timbre/instrument configuration data for AdLib FM synthesis
- Provide a 256-entry lookup table for instrument patch definitions
- Define synthesis parameters (envelope, waveform, feedback) for each instrument slot

## External Dependencies
- None; self-contained data definitions only.


# audiolib/source/linklist.h
## File Purpose
Provides a collection of macro-based utilities for managing doubly-linked circular lists in C. This generic, type-agnostic approach avoids code duplication by parameterizing node types and pointer field names.

## Core Responsibilities
- Create and initialize empty linked lists
- Add, remove, and move nodes within lists
- Transfer entire lists between roots
- Reverse list node ordering
- Insert nodes in sorted order
- Check list state (empty/non-empty)
- Deallocate list memory

## External Dependencies
- `SafeMalloc`, `SafeFree` (memory management—defined elsewhere)
- C++ compatible via `extern "C"` wrapper

# audiolib/source/ll_man.c
## File Purpose
Implements generic doubly-linked list management for an audio library. Provides functions to insert and remove nodes from lists using offset-based pointer arithmetic, with optional memory locking for DOS real-mode environments.

## Core Responsibilities
- Insert nodes at the head of a doubly-linked list
- Remove nodes from a doubly-linked list
- Lock critical linked-list functions in memory (DOS real-mode requirement)
- Unlock memory regions when no longer needed
- Support type-generic linked-list operations via macro-based offset calculations

## External Dependencies
- `<stddef.h>`: Standard C definitions (size_t, NULL).
- **ll_man.h**: Declares public API and macros (LL_AddToHead, LL_AddToTail, LL_Remove).
- **dpmi.h**: DOS Protected Mode Interface declarations for memory locking (DPMI_LockMemoryRegion, DPMI_UnlockMemoryRegion — defined elsewhere).

# audiolib/source/ll_man.h
## File Purpose
Public header for linked list management routines. Provides generic doubly-linked list operations via type-safe macros and underlying functions, supporting dynamic node insertion/removal and memory locking for audio subsystem data structures.

## Core Responsibilities
- Define error codes for linked list operations
- Provide generic node add/remove functions working with offset-based field access
- Expose convenience macros for type-safe head/tail insertion and removal
- Support memory locking/unlocking during linked list manipulation
- Define minimal list container (start/end pointer pair)

## External Dependencies
- Assumes client code defines structs with `next` and `prev` pointers for intrusive linked list integration
- No external includes visible; self-contained interface

# audiolib/source/memcheck.h
## File Purpose
Header file placeholder containing only copyright and GPL 2.0 licensing information. No functional code or declarations are present in the provided file.

## Core Responsibilities
- None (header is empty except for license boilerplate)

## External Dependencies
None

---

**Note:** This file contains only a copyright header (Apogee Software, 1994–1995) and GPL 2.0 license text. The actual header interface (if any) is not shown in the provided content. If there is substantive code in this file, it may not have been included in the excerpt.

# audiolib/source/midi.c
## File Purpose
Core MIDI song playback engine. Loads MIDI files, parses and interprets MIDI events (note on/off, control changes, tempo/time-signature meta events), manages playback state (play/pause/stop/seek), implements volume control, and supports EMIDI extensions (track inclusion/exclusion, looping contexts).

## Core Responsibilities
- Parse MIDI file format (header validation, track enumeration, event data streams)
- Schedule and execute tick-based event interpretation via task manager
- Route MIDI commands to sound device via callback function table (midifuncs)
- Maintain global playback state: active/loaded flags, position counters (ticks, beats, measures, time)
- Manage per-channel and master volume with user-level overrides
- Support EMIDI features: context switching, loop points, track filtering, dynamic program/volume changes
- Provide seek operations (by tick, time, or measure/beat/tick)
- Lock/unlock memory for DOS real-mode interrupt safety

## External Dependencies
- **stdlib.h, time.h, dos.h, string.h**: Standard C
- **sndcards.h**: Sound card enumeration (SoundBlaster, GenMidi, Awe32, UltraSound, etc.)
- **interrup.h**: Interrupt disable/restore (inline asm)
- **dpmi.h**: DOS Protected Mode Interface (memory locking)
- **standard.h**: Type definitions
- **task_man.h**: Task scheduler (TS_ScheduleTask, TS_Terminate, TS_SetTaskRate, TS_Dispatch)
- **ll_man.h**: Linked list (included, use not visible)
- **usrhooks.h**: Memory allocation (USRHOOKS_GetMem, USRHOOKS_FreeMem)
- **music.h**: MIDI error codes and constants (defined elsewhere)
- **_midi.h**: Internal definitions—likely macros (GET_NEXT_EVENT, GET_MIDI_COMMAND, GET_MIDI_CHANNEL), track struct, MIDI event constants, EMIDI codes
- **midi.h**: Public API (user-facing declarations)
- **debugio.h**: Debug output (included but unused)

**Defined Elsewhere**:
- MUSIC_SoundDevice (global sound card type)
- MIDI_* error/status constants
- MIDI event command type constants (MIDI_NOTE_ON, MIDI_CONTROL_CHANGE, etc.)
- Meta event type constants (MIDI_TEMPO_CHANGE, MIDI_END_OF_TRACK, MIDI_TIME_SIGNATURE)
- EMIDI_* extension constants (EMIDI_LOOP_START, EMIDI_CONTEXT_START, etc.)
- EMIDI_AffectsCurrentCard(c2, type) function
- EMIDI_NUM_CONTEXTS, EMIDI_INFINITE, EMIDI_END_LOOP_VALUE
- MIDI_MaxVolume, GENMIDI_DefaultVolume
- NUM_MIDI_CHANNELS (16)
- MAX_FORMAT (max MIDI format version)
- TIME_PRECISION (fixed-point shift)
- RELATIVE_BEAT(measure, beat, tick) macro

# audiolib/source/midi.h
## File Purpose
Public header for MIDI.C declaring the interface for MIDI song file playback and control. Defines error codes, callback function structure, and API for playing, pausing, stopping, and configuring MIDI music during gameplay.

## Core Responsibilities
- Define MIDI error codes and return status enumeration
- Declare callback function interface (`midifuncs` struct) for MIDI event routing to synthesis/output engine
- Provide playback control (play, pause, stop, continue, query playing status)
- Provide tempo and position control (set/get tempo, seek by ticks/time/measure-beat-tick)
- Provide volume control (global and per-channel)
- Provide MIDI system configuration (patch mapping, context, loop flags)
- Provide resource lifecycle management (lock/unlock memory, load timbres, reset)

## External Dependencies
- **Defined elsewhere**: `songposition` struct (type used in position/length queries)
- **Callback interface consumers**: Synthesis engine or hardware output driver that implements `midifuncs` callbacks (NoteOn, NoteOff, ControlChange, etc.)

# audiolib/source/mpu401.c
## File Purpose
Low-level MIDI driver for MPU401-compatible sound cards. Provides functions to send MIDI events (note on/off, control changes, etc.) and manage device initialization, UART mode configuration, and real-time memory locking for DOS-era game audio.

## Core Responsibilities
- Send raw MIDI bytes with configurable I/O delay polling
- Construct and transmit complete MIDI channel messages (note on/off, aftertouch, control changes, program changes, pitch bend)
- Reset and detect MPU401 device; switch to UART mode
- Poll device status port for readiness before I/O operations
- Lock/unlock critical code sections into physical memory for deterministic real-time MIDI playback
- Read user configuration (MPUDELAY override) at initialization

## External Dependencies
- **`<conio.h>`** — `inp()`, `outp()` for I/O port access.
- **`<dos.h>`** — DOS-era system headers (platform-specific).
- **`<stdio.h>`, `<stdlib.h>`** — Standard C (used for `atol()`).
- **`dpmi.h`** — DPMI memory lock/unlock functions (`DPMI_LockMemoryRegion()`, `DPMI_Lock()`, etc.); enum `DPMI_Errors`.
- **`user.h`** — `USER_GetText()` to query user configuration.
- **`mpu401.h`** — Constants and extern declarations (`MPU_BaseAddr`, `MPU_Delay`, MIDI status/command macros).

# audiolib/source/mpu401.h
## File Purpose
Header file defining the interface to an MPU-401 MIDI hardware device controller. The MPU-401 was a standard MIDI interface on PC sound cards in the 1990s. Provides constants, error codes, and function prototypes for initializing the device, transmitting MIDI events, and managing real-time memory access.

## Core Responsibilities
- Define MPU-401 hardware constants (default I/O address, command codes, status flags)
- Declare error/status codes for device operations
- Export device initialization and reset functions
- Provide low-level MIDI transmission (raw bytes and structured MIDI events)
- Export memory-locking utilities for real-time MIDI interrupt safety

## External Dependencies
- Standard C integers (`int`, `unsigned`)
- DPMI (DOS Protected Mode Interface) – implied for memory locking (defined elsewhere)
- MPU-401 hardware I/O controller (typically at 0x330)

# audiolib/source/multivoc.c
## File Purpose

Core multi-voice audio mixing engine for DOS-era Sound Blaster and compatible sound cards. Manages voice allocation, mixing, playback across multiple audio formats (VOC, WAV, RAW), and sound effects like reverb and 3D panning using interrupt-driven DMA-based circular buffers.

## Core Responsibilities

- Initialize and manage multi-voice playback system with hardware sound card drivers
- Allocate and manage a pool of voice nodes with priority-based preemption
- Mix multiple active voices into circular DMA buffers at regular intervals
- Parse and playback audio in VOC, WAV, and raw formats with looping support
- Support demand-fed (callback-based) audio for streaming playback
- Apply per-voice volume, pitch, stereo pan, and reverb effects via lookup tables
- Handle sound card hardware abstractions (Sound Blaster, Gravis UltraSound, Pro Audio Spectrum, etc.)
- Manage error codes and user callbacks when voices finish playback
- Implement recording mode for supported sound cards
- Lock/unlock critical real-time code and data to physical memory (DPMI)

## External Dependencies

**System / Low-Level**:
- `<stdlib.h>`, `<string.h>`, `<dos.h>`, `<time.h>`, `<conio.h>` — Standard C and DOS
- **dpmi.h** — DOS Protected Mode Interface (memory locking, DOS memory allocation)
- **usrhooks.h** — User-controlled memory allocation hooks
- **interrup.h** — CPU interrupt enable/disable
- **dma.h** — DMA controller access (get current position)

**Linked List Support**:
- **linklist.h** — Linked list macros (LL_SortedInsertion, LL_Add, LL_Remove, etc.)

**Sound Card Drivers** (multiple cards supported):
- **sndcards.h** — Sound card type enums (SoundBlaster, UltraSound, ProAudioSpectrum, etc.)
- **blaster.h** — Sound Blaster driver (BLASTER_Init, BLASTER_BeginBufferedPlayback, BLASTER_SetMixMode, BLASTER_DMAChannel, BLASTER_GetPlaybackRate, BLASTER_GetCurrentPos, BLASTER_StopPlayback)
- **guswave.h** — Gravis UltraSound driver (GUSWAVE_Init, GUSWAVE_StartDemandFeedPlayback, GUSWAVE_KillAllVoices, GUSWAVE_Shutdown)
- **sndscape.h** — Ensoniq SoundScape driver (SOUNDSCAPE_Init, SOUNDSCAPE_BeginBufferedPlayback, SOUNDSCAPE_SetMixMode, SOUNDSCAPE_DMAChannel, SOUNDSCAPE_StopPlayback, SOUNDSCAPE_GetCurrentPos)
- **pas16.h** — Pro Audio Spectrum driver (PAS_Init, PAS_BeginBufferedPlayback, PAS_SetMixMode, PAS_DMAChannel, PAS_GetPlaybackRate, PAS_StopPlayback, PAS_GetCurrentPos)
- **sndsrc.h** — Sound Source driver (SS_Init, SS_BeginBufferedPlayback, SS_SetMixMode, SS_SampleRate, SS_StopPlayback)

**Audio Processing**:
- **pitch.h** — Pitch/frequency scaling (PITCH_GetScale, PITCH_LockMemory, PITCH_UnlockMemory)
- **multivoc.h**, **_multivc.h** — Local header (VoiceNode definition, MV_Mix8BitMono, MV_Mix8BitStereo, MV_Mix16BitMono, MV_Mix16BitStereo variants, MV_8BitReverb, MV_16BitReverb, MV_8BitReverbFast, MV_16BitReverbFast, ClearBuffer_DW)

**Debug Output**:
- **debugio.h** — Debug output functions

---

**Key Macros Defined Locally**:
- `RoundFixed(fixedval, bits)` — Fixed-point rounding
- `IS_QUIET(ptr)` — Check if volume pointer is zero (silent)
- `MV_SetErrorCode(status)` — Update error code
- `MV_LockStart` / `MV_LockEnd` — Memory-lock region markers for MV_Mix and related functions

# audiolib/source/multivoc.h
## File Purpose
Public API header for MULTIVOC, a multi-voice software audio mixer library. Defines the interface for initializing the audio system, managing voice channels, controlling playback parameters, and playing various audio formats (raw, WAV, VOC).

## Core Responsibilities
- Define error codes and global error state for audio operations
- Declare voice lifecycle functions (allocate, play, kill)
- Declare playback control functions (pitch, pan, reverb, 3D audio)
- Declare audio format loaders (WAV, VOC, raw data)
- Declare mixing mode and volume configuration
- Declare system initialization and memory management

## External Dependencies
- No standard library includes visible
- Callback functions supplied as function pointers (`void (*function)(...)`)
- Hardware abstraction via soundcard parameter in MV_Init
- All implementation details defined elsewhere (MULTIVOC.C)

# audiolib/source/music.c
## File Purpose
Device-independent music playback manager for the Apogee Sound System. Provides high-level MIDI music control abstraction over multiple sound card drivers (SoundBlaster, AdLib, GUS, etc.), handling initialization, playback, volume control, and fade effects.

## Core Responsibilities
- Initialize sound devices and select appropriate driver implementations
- Provide unified MIDI playback API (play, pause, stop, seek)
- Manage volume control and implement fade-in/fade-out effects via background task
- Detect and report device initialization errors
- Route MIDI events through device-specific function pointers
- Manage song playback state (looping, context, position)

## External Dependencies
- **task_man.h:** Task scheduling (TS_ScheduleTask, TS_Terminate, TS_Dispatch)
- **sndcards.h:** Sound card enum definitions
- **midi.h:** MIDI playback layer (MIDI_PlaySong, MIDI_SetVolume, MIDI_SetMidiFuncs, etc.)
- **al_midi.h, pas16.h, blaster.h, gusmidi.h, mpu401.h, awe32.h, sndscape.h:** Device-specific drivers (defined elsewhere)
- **ll_man.h:** Memory locking (LL_LockMemory, LL_UnlockMemory)
- **user.h:** Command-line parameter checking (USER_CheckParameter)

# audiolib/source/music.h
## File Purpose
Public API header for the MUSIC.C module. Declares functions and types for MIDI/music playback control, including song loading, playback state management, volume control, and position seeking in the Apogee Sound System (ASS).

## Core Responsibilities
- Initialize and shut down the music/MIDI subsystem
- Load and play MIDI song data with loop control
- Control playback state (play, pause, stop, continue)
- Manage master and per-channel MIDI volume
- Seek to specific song positions (ticks, milliseconds, measures/beats)
- Apply fade effects to volume over time
- Configure MIDI channel routing and timbre banks
- Report music system errors

## External Dependencies
- **Include**: `sndcards.h` (defines soundcardnames enum)
- **Defined elsewhere**: All function implementations in MUSIC.C; hardware abstraction and MIDI I/O handled by sound card drivers

# audiolib/source/mv1.c
## File Purpose
Core multichannel digitized sound mixer for Sound Blaster and compatible cards. Implements real-time audio mixing of multiple voices (channels) into a double-buffered output, with support for mono/stereo and 8-bit/16-bit formats. Handles voice allocation, 3D positioning, pitch/pan control, and VOC format playback.

## Core Responsibilities
- **Voice Management**: Allocate/deallocate voice nodes from a pool, with priority-based voice stealing for overcommitment
- **Audio Mixing**: Mix multiple voices into a mono or stereo buffer at 8-bit or 16-bit resolution using fixed-point rate scaling
- **Playback Control**: Manage double-buffering scheme, interrupt-driven buffer service, and integration with hardware sound cards
- **Spatial Audio**: 3D panning and distance attenuation via lookup tables; pitch scaling via external pitch module
- **Format Handling**: Parse VOC format sound blocks, manage looping, and detect format variants
- **Volume & Clipping**: Precalculate volume lookup tables per voice, apply soft clipping via harsh clip tables

## External Dependencies

**Direct includes / imports**:
- `stdlib.h`: Standard C library (malloc, etc.; not used directly in this file but likely in implementations)
- `dpmi.h`: DPMI (DOS Protected Mode Interface) for memory locking and DOS memory allocation
- `usrhooks.h`: User-provided memory allocation hooks
- `interrup.h`: Interrupt enable/disable macros
- `ll_man.h`: Linked list management macros
- `sndcards.h`: Sound card driver abstraction (defines SoundBlaster, Awe32, ProAudioSpectrum, etc.)
- `blaster.h`: Sound Blaster driver API
- `sndsrc.h`: Sound Source driver API (conditional)
- `pas16.h`: Pro Audio Spectrum driver API
- `pitch.h`: Pitch scaling module
- `multivoc.h`: Public header (client-facing API)
- `_multivc.h`: Private header (likely defines VoiceNode, VList, Pan, volume table types)

**External symbols used but not defined here**:
- `BLASTER_Init()`, `BLASTER_BeginBufferedPlayback()`, `BLASTER_StopPlayback()`, `BLASTER_Shutdown()`, `BLASTER_GetPlaybackRate()`, `BLASTER_SetMixMode()`, `BLASTER_Error`, `BLASTER_ErrorString()`
- `PAS_Init()`, `PAS_BeginBufferedPlayback()`, `PAS_StopPlayback()`, `PAS_Shutdown()`, `PAS_GetPlaybackRate()`, `PAS_SetMixMode()`, `PAS_Error`, `PAS_ErrorString()`
- `SS_Init()`, `SS_BeginBufferedPlayback()`, `SS_StopPlayback()`, `SS_Shutdown()`, `SS_SetMixMode()`, `SS_Error`, `SS_ErrorString()`, `SS_SampleRate`
- `ClearBuffer_DW()`: Clear buffer by 32-bit word (likely optimized memset variant)
- `PITCH_Init()`, `PITCH_GetScale()`, `PITCH_LockMemory()`, `PITCH_UnlockMemory()`: Pitch module
- `USRHOOKS_GetMem()`, `USRHOOKS_FreeMem()`: User-provided memory allocation
- `DPMI_GetDOSMemory()`, `DPMI_FreeDOSMemory()`, `DPMI_LockMemory()`, `DPMI_LockMemoryRegion()`, `DPMI_Unlock()`, `DPMI_UnlockMemory()`, `DPMI_UnlockMemoryRegion()`: DPMI interface
- `DisableInterrupts()`, `RestoreInterrupts()`: Inline interrupt control
- `LL_AddToHead()`, `LL_AddToTail()`, `LL_Remove()`: Linked list macros
- `MIX_VOLUME()`: Volume scaling macro
- `max()`, `min()`: Common macros

# audiolib/source/mv_mix.asm
## File Purpose
Low-level x86-32 assembly implementation of audio sample mixing routines. Provides four functions to mix audio at different bit depths (8-bit, 16-bit) and channel configurations (mono, stereo), using self-modifying code for runtime parameter injection.

## Core Responsibilities
- Resample audio from source buffer using fractional position tracking and rate scaling
- Apply per-channel volume translation via lookup tables
- Accumulate (mix) resampled samples with destination output buffer
- Clip results to valid audio ranges (harsh clipping for 8-bit via table, explicit bounds-checking for 16-bit)
- Update global mixing state (destination write position and playback position) after each frame
- Use self-modifying code patching to inject runtime parameters (volume tables, sample offsets, rates) into inner loops

## External Dependencies
- **External symbols (EXTRN):**
  - `_MV_HarshClipTable` — 8-bit clipping lookup table (offset ±128 for signed range)
  - `_MV_MixDestination`, `_MV_MixPosition` — mixing state variables
  - `_MV_LeftVolume`, `_MV_RightVolume` — channel volume lookup tables
  - `_MV_SampleSize`, `_MV_RightChannelOffset` — format configuration constants
- **Assembler directives:** `IDEAL` mode (Borland TASM), `p386` (32-bit), `MODEL flat` (flat memory), `MASM` compatibility
- **No includes or inter-file function calls** — self-contained inner loops with code patching

# audiolib/source/mv_mix1.asm
## File Purpose
Low-level optimized x86 assembly mixer for 8-bit mono audio samples. Uses self-modifying code and lookup tables to rapidly apply volume scaling and clipping during audio buffer mixing.

## Core Responsibilities
- Mix 8-bit mono audio samples with fractional sample positioning
- Apply volume transformation via lookup table
- Apply harsh clipping via lookup table to prevent distortion
- Self-patch code with runtime parameters for zero-overhead parameter passing
- Process fixed buffer size (256 samples) per invocation

## External Dependencies
None (standalone assembly).

# audiolib/source/mv_mix16.asm
## File Purpose
Low-level x86 assembly implementation of audio sample mixing for a game engine. Provides optimized mixing routines that combine source audio into a destination buffer with volume scaling, sample rate conversion, and overflow clipping. Supports 8-bit and 16-bit samples in mono and stereo configurations.

## Core Responsibilities
- Mix audio samples from source buffer into destination buffer with volume translation
- Apply per-channel volume via lookup table (8-bit) or direct scaling (16-bit)
- Perform sample rate conversion using 16.16 fixed-point fractional position arithmetic
- Prevent overflow via harsh clipping (lookup table for 8-bit, direct comparison for 16-bit)
- Support dual sample processing per loop iteration for efficiency
- Use runtime self-modifying code ("patching") to inject configuration values into inner loops

## External Dependencies
- **TASM directives:** IDEAL, MODEL flat, MASM, ALIGN, PROC, PUBLIC, EXTRN, ENDS, END
- **External symbols:** `_MV_HarshClipTable`, `_MV_MixDestination`, `_MV_MixPosition`, `_MV_LeftVolume`, `_MV_RightVolume`, `_MV_SampleSize`, `_MV_RightChannelOffset` (all defined elsewhere, likely in C runtime)
- **No internal calls:** Pure assembly with no dependencies on other functions in this file

# audiolib/source/mv_mix2.asm
## File Purpose
Hand-optimized x86 assembly implementation of 8-bit mono audio sample mixing. Processes 256 samples per invocation, applying volume translation and harsh clipping to prevent distortion during real-time audio mixing.

## Core Responsibilities
- Fast inner-loop mixing of 8-bit mono audio samples
- Applies volume translation via lookup table
- Applies harsh clipping via lookup table to prevent overflow
- Handles fractional sample position tracking for potential resampling
- Uses self-modifying code for runtime table pointer inlining
- Processes samples in pairs (two per iteration) for throughput optimization

## External Dependencies
- **Includes**: None visible (pure x86 assembly).
- **External symbols**: Invoked as a callable function (likely from C/C++ audio mixer); table pointers and buffers supplied by caller.
- **Assembler**: TASM (Turbo Assembler) syntax (`.386`, `.MODEL flat`, `SEGMENT`, `PROC`/`ENDP`).

# audiolib/source/mv_mix3.asm
## File Purpose
Hand-optimized x86-32 assembly audio mixing routines for the sound engine. Provides fast mixing of 8-bit and 16-bit audio samples in mono and stereo configurations with sample-rate conversion via fractional position tracking and real-time clipping.

## Core Responsibilities
- Mix source audio samples into a destination buffer with volume scaling
- Perform sample-rate conversion using fractional position advancement
- Apply per-sample clipping to prevent overflow
- Support multiple audio formats: 8-bit/16-bit, mono/stereo, and 1-channel layouts
- Use self-modifying code patterns to inject runtime parameters (volume tables, clip tables, sample rates) into instruction immediates

## External Dependencies
- **Notable includes / imports:** None (pure assembly)
- **Defined elsewhere:** 
  - Volume lookup tables (passed as pointers)
  - Harsh clip tables (passed as pointers, offset by +128)
  - Source and destination audio buffers (passed as pointers)
  - All parameters are caller-provided; no global variables referenced

# audiolib/source/mv_mix4.asm
## File Purpose
Low-level x86 assembly audio mixing routines that blend source audio samples into a destination buffer with volume scaling and sample-rate conversion. Supports 8-bit and 16-bit PCM with mono, stereo, and single-channel layouts using runtime code patching for performance optimization.

## Core Responsibilities
- Mix source audio samples into destination buffer with per-sample volume translation
- Handle sample-rate conversion via fractional position tracking (fixed-point arithmetic)
- Apply harsh clipping to prevent audio distortion (table-based for 8-bit, inline comparisons for 16-bit)
- Support six audio format variants: 8/16-bit × mono/stereo/1-channel
- Optimize hot loops through self-modifying code injection of runtime parameters

## External Dependencies
- **Assembler directives**: `.386`, `.MODEL flat`, `USE32` (80386 protected-mode 32-bit segments)
- **External symbols** (defined elsewhere in audio library):
  - `_MV_HarshClipTable` – lookup table for 8-bit clipping
  - `_MV_MixDestination` – output buffer write pointer
  - `_MV_MixPosition` – fractional sample playback position
  - `_MV_LeftVolume` – mono/left-channel volume table
  - `_MV_RightVolume` – right-channel volume table

# audiolib/source/mv_mix5.asm
## File Purpose
Low-level assembly implementation of high-performance audio sample mixing functions. Provides six variants for mixing 8-bit and 16-bit audio in mono, stereo, and single-channel configurations with real-time volume control and sample-rate interpolation.

## Core Responsibilities
- Mix source audio samples into destination buffer with rate-based interpolation
- Apply volume scaling via pre-computed lookup tables (separate for left/right channels)
- Clip/clamp mixed samples to prevent distortion (harsh clip table for 8-bit, min/max clamping for 16-bit)
- Dynamically patch instruction immediates at runtime for zero-overhead parameter passing
- Process multiple samples per loop iteration (typically two at a time) for throughput
- Maintain and update global playback position and output buffer pointers

## External Dependencies
- **Externals** (defined elsewhere): `_MV_HarshClipTable`, `_MV_MixDestination`, `_MV_MixPosition`, `_MV_LeftVolume`, `_MV_RightVolume`
- No #include or imports; pure 32-bit x86 assembly

**Technical notes**: Self-modifying code patterns use OFFSET patching to inject runtime values into immediates, avoiding register pressure. Comment "convice tasm to modify code" suggests TASM assembler compatibility workaround. Hardcoded placeholder immediates (12345678h) are replaced at runtime.

# audiolib/source/mv_mix6.asm
## File Purpose
High-performance x86 assembly audio mixing kernels for the audio library. Implements six specialized functions to mix audio samples at different bit depths (8-bit, 16-bit) and channel configurations (mono, stereo, 1-channel interleaved). Uses self-modifying code and tight loop optimization for real-time audio performance.

## Core Responsibilities
- Mix source audio samples into a destination buffer via fractional sample position tracking
- Apply per-channel volume scaling using lookup tables (8-bit) or direct scaling (16-bit)
- Resample audio by advancing a fractional position counter by a per-sample rate increment
- Clip/clamp mixed samples to valid output ranges (via lookup table for 8-bit, conditional branches for 16-bit)
- Update global mixing state (destination write pointer, current playback position)
- Process samples in pairs (8-bit variants) or one at a time (stereo/16-bit) for instruction-level parallelism

## External Dependencies
- **External symbols**:
  - `_MV_HarshClipTable`: Defined elsewhere; used for 8-bit sample clipping via lookup
  - `_MV_MixDestination`: Global output buffer cursor
  - `_MV_MixPosition`: Global fractional sample position
  - `_MV_LeftVolume`: Defined elsewhere; 16-entry lookup table for left-channel volume
  - `_MV_RightVolume`: Defined elsewhere; 16-entry lookup table for right-channel volume

- **Assembler syntax**: TASM IDEAL mode; directives `p386`, `MODEL flat`, `MASM` compatibility; `ALIGN 4` loop labels for instruction cache alignment

# audiolib/source/mvreverb.asm
## File Purpose
Implements audio reverb/echo effects processing routines in x86 assembly for the audio mixing engine. Provides both table-lookup based reverb (with volume attenuation) and shift-based fast reverb operations for 8-bit and 16-bit PCM audio samples.

## Core Responsibilities
- Process 16-bit and 8-bit audio samples through reverb effects
- Apply volume table transformations to attenuate/transform sample data
- Implement fast reverb path using arithmetic right shifts with rounding
- Copy and mix audio samples from source to destination buffers
- Support self-modifying code for runtime shift amount specialization

## External Dependencies
- None: pure x86 assembly with no external symbols or imports.

# audiolib/source/myprint.c
## File Purpose

Implements low-level text output and formatting for DOS-era VGA text mode (80×24 characters). Provides direct video memory manipulation (0xb0000), text positioning, frame drawing, and printf-like formatted output with automatic scrolling.

## Core Responsibilities

- Direct character rendering to VGA text memory with color attributes
- Screen cursor positioning and line-based text output
- Rectangular text fill and frame drawing with border characters
- Printf-like format string processing (%d, %s, %u, %x specifiers)
- Automatic screen scrolling when output exceeds display bounds
- Newline/carriage return handling with line clearing

## External Dependencies

- `<stdio.h>, <stdarg.h>, <stdlib.h>`: standard C headers.
- `itoa(int, char*, int)` (stdlib): convert signed int to string.
- `ultoa(unsigned long, char*, int)` (stdlib): convert unsigned long to string.
- `myprint.h`: local header (COLORS enum, frame type macros, function declarations).

# audiolib/source/myprint.h
## File Purpose
This header declares text rendering and formatting primitives for screen output. It provides colored text drawing, cursor positioning, and printf-like output functions, suggesting it's a display utility module used across the engine despite being in the audiolib directory.

## Core Responsibilities
- Define standard 16-color palette (DOS/early graphics colors)
- Declare character-level drawing primitives (DrawText, TextBox, TextFrame)
- Provide cursor-based text output functions (myputch, printstring)
- Declare formatted printing with variable arguments (myprintf)
- Support arbitrary radix integer conversion (printunsigned)
- Define frame decoration constants (SINGLE_FRAME, DOUBLE_FRAME)

## External Dependencies
None (header-only declarations). No includes. Uses COLORS enum values and frame constants defined locally.

# audiolib/source/newgf1.h
## File Purpose
Header file defining the public interface to the Gravis Ultrasound GF1 audio card driver library. Provides structures, constants, macros, and function prototypes for low-level control of GF1 hardware including DMA transfers, voice allocation, MIDI playback, digital audio streaming, patch loading, and waveform management.

## Core Responsibilities
- Define error codes and hardware status constants (DMA, IRQ, card detection)
- Define bit flags for DMA control, MIDI signaling, digital playback modes, and patch properties
- Define on-disk patch file structures (header, instrument, layer, waveform metadata)
- Define runtime audio structures (patch, waveform, sound instance, DMA buffer)
- Declare hardware initialization and detection functions
- Declare DMA and GF1 DRAM memory management functions
- Declare voice allocation and priority management functions
- Declare digital audio playback control functions (start, stop, pause, streaming)
- Declare MIDI note triggering and control change functions
- Declare patch/waveform loading and management functions
- Declare timer and callback registration functions

## External Dependencies
- Platform-specific compiler macros (BORLANDC, _MSC_VER) for far pointer qualification (RFAR)
- GF1 hardware memory model (DMA, DRAM, port I/O operations)
- File I/O functions (gf1_open, gf1_read, gf1_close_file) for patch loading
- Callback-based architecture (function pointers) for DMA completion, voice stealing, MIDI events, timers
- MIDI protocol constants and control change definitions

# audiolib/source/oldtimbr.c
## File Purpose
Provides OPL chip FM synthesis timbre (instrument) presets for a retro game audio library. Contains configuration data for 174 instrument timbres and a percussion mapping table for MIDI-to-sound routing.

## Core Responsibilities
- Define TIMBRE struct for FM operator parameters (SAVEK, envelope, waveform, feedback)
- Define DRUM_MAP struct for percussion note-to-timbre mapping
- Supply PercussionTable array mapping all 128 MIDI percussion notes to drum timbres
- Supply ADLIB_TimbreBank with 174 preset instrument configurations

## External Dependencies
- No includes
- Assumed consumed by audio driver code elsewhere in `audiolib/` that configures Yamaha OPL FM synthesizers

# audiolib/source/pas16.c
## File Purpose
Low-level hardware driver for Pro AudioSpectrum (PAS) sound cards on DOS/protected-mode systems. Manages interrupt handlers, DMA transfers, hardware state, and mixer control via a loadable real-mode driver (MVSOUND.SYS). Bridges protected-mode (386+) code to real-mode hardware and driver using DPMI calls.

## Core Responsibilities
- **Driver detection & initialization**: Verify MVSOUND.SYS driver presence, query DMA/IRQ configuration, auto-detect card port address.
- **Interrupt management**: Install/uninstall interrupt handlers for DMA completion, manage interrupt controller masks, chain to old handlers.
- **DMA setup & monitoring**: Configure DMA channels, manage circular DMA buffers, track playback position, coordinate with interrupt-driven transfers.
- **Hardware configuration**: Program sample rate timers, buffer counters, mix mode (mono/stereo/8/16-bit), audio filters, cross-channel controls.
- **Volume control**: PCM and FM mixer volume management via real-mode driver function calls.
- **State preservation**: Save/restore original hardware state and mixer settings across init/shutdown.
- **Protected-mode support**: Lock/unlock memory for real-mode access, allocate conventional stack for interrupt handler, use DPMI for real-mode calls.

## External Dependencies
- **C Standard Library:** `dos.h`, `conio.h` (x86 DOS), `stdlib.h`, `stdio.h`, `string.h`.
- **Custom DPMI Layer:** `dpmi.h` — DPMI 0x31 interrupt (real-mode calls, memory allocation).
- **Custom DMA Layer:** `dma.h` — DMA controller setup and state.
- **Interrupt Management:** `interrup.h`, `irq.h` — Interrupt flag control and high-IRQ vector setup.
- **Public Interface:** `pas16.h`, `_pas16.h` (private definitions).
- **Hardware Driver:** MVSOUND.SYS (real-mode DOS driver) — accessed via software interrupt (0x??) for mixer and state queries.
- **Symbols defined elsewhere:** `CalcTimeInterval`, `CalcSamplingRate`, `PAS_TestAddress` (used but not shown), `RECORD`, `PLAYBACK`, `MONO_8BIT`, `STEREO_16BIT`, `SampleBufferInterruptFlag`, `InterruptStatus`, `InterruptControl`, etc. (likely in `_pas16.h`).

# audiolib/source/pas16.h
## File Purpose
Public header for the PAS16 audio driver implementation. Declares the interface for controlling a ProAudio Spectrum 16 sound card, including buffered playback/recording, volume control, and driver lifecycle management. Intended for games and audio applications running on legacy DOS/Windows 9x systems with PAS16 hardware.

## Core Responsibilities
- Define error codes for all PAS16 operations
- Expose configuration macros (sample rates, mix modes, DMA, IRQ limits)
- Declare buffered playback and recording functions with callback support
- Provide volume control for PCM and FM synthesis
- Manage audio format negotiation (sampling rate, channels, bit depth)
- Support driver initialization, shutdown, and memory locking for real-time safety

## External Dependencies
- References `STEREO_16BIT` and `MONO_8BIT` constants (defined elsewhere, likely in a mixing/audio library header)
- Implicitly depends on DOS/Windows extended memory and DMA infrastructure
- No explicit `#include` directives in this file

# audiolib/source/pcfx.c
## File Purpose
Low-level PC speaker sound effects driver for rendering mono audio samples created by Muse. Provides playback control, single-voice voice management, and volume control via the internal PC speaker using I/O port writes. Tightly integrated with the task scheduler for real-time sample output.

## Core Responsibilities
- Manage single-voice mono playback with priority preemption
- Render audio samples to PC speaker via I/O ports (0x61, 0x42, 0x43)
- Support dual sample formats: pitch-indexed lookup table or raw 16-bit PCM samples
- Integrate with task manager to output samples at regular intervals
- Track playback state and handle completion callbacks
- Lock/unlock memory regions for real-time safe operation in protected mode
- Provide error reporting and volume control (0–255 range)

## External Dependencies
- **Includes:** `<dos.h>`, `<stdlib.h>`, `<conio.h>` (legacy DOS/Watcom headers)
- **Local headers:** dpmi.h (memory locking), task_man.h (scheduling), interrup.h (interrupt control), pcfx.h (public API & PCSound struct)
- **I/O ports (hardware):** 0x61 (speaker control), 0x42, 0x43 (timer frequency)
- **Symbols defined elsewhere:**
  - DPMI_LockMemoryRegion, DPMI_Lock, DPMI_UnlockMemoryRegion, DPMI_Unlock (memory protection)
  - TS_ScheduleTask, TS_Dispatch, TS_Terminate (task scheduling)
  - DisableInterrupts, RestoreInterrupts (interrupt control)
  - max, min macros (stdlib)

# audiolib/source/pcfx.h
## File Purpose
Public header for a PC sound effects (PCFX) library providing digital audio playback with voice management and priority-based mixing. Defines the engine-facing API for sound initialization, playback control, and shutdown.

## Core Responsibilities
- Define error codes for PCFX operations
- Declare the PCSound data structure for sound effect data
- Declare public API for sound playback and voice management
- Provide volume control and callback mechanism for sound completion
- Define memory locking interface for audio resources

## External Dependencies
- No includes visible (header guard only)
- Implementation referenced as ADLIBFX.C per header comment
- Symbols for audio driver interaction not defined in this file

# audiolib/source/pitch.c
## File Purpose
Provides pitch scaling calculations for audio playback. Uses a precomputed lookup table to map MIDI-style pitch offsets (in cents) to fixed-point scale factors for playback rate adjustment. Includes DOS/DPMI memory locking to ensure deterministic real-time audio performance.

## Core Responsibilities
- Calculate fixed-point pitch scale factors from pitch offsets in cents
- Supply a precomputed 12-note × 25-detune lookup table for efficient pitch calculation
- Lock/unlock pitch code and data in physical memory for real-time audio without page swaps
- Support arbitrary pitch offsets by decomposing into octave, semitone, and detune components

## External Dependencies
- **dpmi.h**: DOS DPMI memory locking (`DPMI_LockMemoryRegion`, `DPMI_UnlockMemoryRegion`, `DPMI_Lock`, `DPMI_Unlock`); error codes.
- **standard.h**: Standard macro definitions (included but unused in this file).
- **pitch.h**: Public interface declarations.
- **stdlib.h**: Standard C library (not directly used).

# audiolib/source/pitch.h
## File Purpose
Public header for the PITCH module, which handles pitch scaling calculations and memory management for audio pitch operations. This is part of the original Apogee Software audio library from 1994.

## Core Responsibilities
- Define pitch operation error codes
- Declare pitch scaling calculation interface
- Manage memory lifecycle (lock/unlock) for pitch data

## External Dependencies
- No explicit includes shown; implementation in PITCH.C
- Note: `PITCH_Init()` is declared but commented out, suggesting initialization may be handled elsewhere or was removed

# audiolib/source/sndcards.h
## File Purpose
Header file defining enumerated types for supported audio device types in the sound library. Provides a centralized list of sound card identifiers and a version constant for the audio subsystem.

## Core Responsibilities
- Enumerate all supported sound card device types (Sound Blaster, Adlib, SoundCanvas, UltraSound, etc.)
- Define the library version string
- Provide shared type definitions for the audio subsystem initialization and device selection

## External Dependencies
- Standard C preprocessor directives only (`#ifndef`, `#define`, `#endif`)
- No external symbol dependencies

**Notes:**
- One enum entry is commented out (`ASS_NoSound`), suggesting prior removal or conditional support
- Device enumeration reflects late 1980s/early 1990s audio hardware (Sound Blaster, Adlib, UltraSound, etc.)
- PC and SoundScape entries may represent fallback/generic options

# audiolib/source/sndscape.c
## File Purpose
Low-level driver for the Ensoniq SoundScape sound card in DOS/DPMI environment. Handles hardware initialization, interrupt-driven DMA playback, and PCM audio configuration via the AD-1848 codec and gate-array chip.

## Core Responsibilities
- Detect and initialize SoundScape hardware from SNDSCAPE.INI configuration file
- Configure DMA channels and IRQ vectors for interrupt-driven audio transfer
- Manage AD-1848 codec register access (sample rate, bit depth, stereo/mono)
- Service sound card interrupts and invoke user callback at half-buffer boundaries
- Lock critical memory regions with DPMI to ensure interrupt handler stability
- Provide public API for playback control (start/stop, rate/mode configuration)
- Handle Ensoniq gate-array signal routing and chip-specific quirks (ODIE/OPUS/MMIC)

## External Dependencies
- **Includes:** `dos.h`, `conio.h`, `stdlib.h`, `stdio.h`, `string.h`, `ctype.h`, `time.h` (DOS runtime)
- **Local headers:** `interrup.h` (DisableInterrupts/RestoreInterrupts), `dpmi.h` (memory locking), `dma.h` (DMA controller), `irq.h` (IRQ setup), `sndscape.h` (public API)
- **External symbols (defined elsewhere):**
  - `DMA_SetupTransfer()`, `DMA_EndTransfer()`, `DMA_VerifyChannel()`, `DMA_GetCurrentPos()`, `DMA_ErrorString()` (dma.c)
  - `IRQ_SetVector()`, `IRQ_RestoreVector()` (irq.c)
  - `DPMI_LockMemoryRegion()`, `DPMI_Lock()`, etc. (dpmi.c)
  - `_dos_getvect()`, `_dos_setvect()`, `_chain_intr()`, `int386()` (DOS/compiler runtime)
  - `inp()`, `outp()` (compiler intrinsics for I/O ports)

# audiolib/source/sndscape.h
## File Purpose
Public header for the Soundscape audio driver module. Declares the interface for initializing, configuring, and controlling a Soundscape sound card for buffered PCM playback with DMA and interrupt-driven callbacks.

## Core Responsibilities
- Audio device initialization and hardware detection (Soundscape-specific)
- Playback rate and mixing mode configuration
- Buffered playback management with DMA transfers
- Callback-driven audio completion events
- Error reporting and hardware capability queries
- MIDI port and IRQ configuration

## External Dependencies
- None visible in header (implementation in sndscape.c likely includes hardware I/O, DMA setup, IRQ handler registration)
- Assumes DOS/x86 real-mode or protected-mode (DPMI) environment with ISA hardware access

# audiolib/source/sndsrc.c
## File Purpose

Low-level driver for the Disney Sound Source, a legacy parallel-port audio device (ca. 1994). Manages buffered playback of 8-bit mono digitized audio at 7 kHz via interrupt-driven I/O port transfers with user callback support.

## Core Responsibilities

- **Hardware detection & initialization**: Locate Sound Source at parallel ports (0x3BC, 0x378, 0x278) with optional Tandy variant support
- **Buffered playback management**: Maintain circular buffer state, track playback position, manage multi-buffer cycling
- **Interrupt-driven transfers**: Service timer interrupts to stream samples to the I/O port at a controlled rate (~438–510 ticks/sec)
- **Callback coordination**: Invoke user function when buffer division transfer completes
- **Memory safety for interrupts**: Lock critical code & data in real memory (DPMI) to prevent page faults during interrupt service
- **Error tracking & reporting**: Provide error codes and human-readable error strings

## External Dependencies

- **dos.h, conio.h**: `inp()`, `outp()` for legacy parallel-port I/O
- **dpmi.h**: `DPMI_LockMemoryRegion`, `DPMI_Lock`, `DPMI_Unlock` for real-mode interrupt safety (prevents page faults)
- **task_man.h**: `task` struct, `TS_ScheduleTask`, `TS_Terminate`, `TS_Dispatch` for timer interrupt scheduling
- **sndcards.h**: `TandySoundSource` enum for variant selection
- **user.h**: `USER_CheckParameter()` for configuration parameter lookup

# audiolib/source/sndsrc.h
## File Purpose
Public header for the SoundSource audio device driver module (SNDSRC.C). Declares the API for controlling the SoundSource parallel-port audio device, including initialization, playback control, and memory management for DMA operations.

## Core Responsibilities
- Define error codes returned by SoundSource operations
- Declare public initialization and shutdown functions
- Declare playback control functions (start, stop, rate control)
- Declare parallel port configuration functions (select port, set callback)
- Define port addresses and device constants for multiple hardware configurations
- Declare memory locking/unlocking for DMA-safe buffers

## External Dependencies
- ANSI C standard library (implicit)
- Parallel port hardware; DPMI for protected-mode memory locking (DOS/DPMI environment)
- Direct hardware I/O to parallel ports (addresses 0x3bc, 0x278, 0x378)
- Tandy sound hardware (legacy support)

# audiolib/source/standard.h
## File Purpose
Standard definitions header for the audio library. Provides common type aliases, utility macros for bit operations and array bounds checking, error code enumerations, and debugging utilities used throughout the codebase.

## Core Responsibilities
- Define standard type aliases (`boolean`, `errorcode`)
- Define standard error codes (`Success`, `Warning`, `FatalError`)
- Provide utility macros for bitwise operations, array handling, and loops
- Provide conditional debugging macros

## External Dependencies
- None. File is self-contained with only standard C preprocessor directives.


# audiolib/source/task_man.c
## File Purpose

Low-level timer task scheduler for DOS x86 systems. Manages periodic task execution via hardware interrupt (INT 8, the 8253 timer). Provides real-time task scheduling with priority ordering, interrupt-safe operation, and optional memory locking for deterministic behavior.

## Core Responsibilities

- Schedule and manage periodic tasks executed from timer interrupt context
- Control the 8253 timer hardware, calculating interrupt rates from task frequencies
- Maintain doubly-linked task list with priority-based insertion and removal
- Provide two interrupt handling modes: disabled-interrupt and re-entrant with interrupt enabled
- Optionally allocate and switch to dedicated interrupt stack (USESTACK mode)
- Lock memory regions to prevent page faults during interrupt handling (LOCKMEMORY mode)
- Activate/deactivate tasks and adjust their execution rates dynamically

## External Dependencies

**Headers:**
- `interrup.h`: DisableInterrupts(), RestoreInterrupts() — inline asm for PUSHFD/POPFD/CLI
- `linklist.h`: LL_SortedInsertion, LL_RemoveNode macros — doubly-linked list operations
- `task_man.h`: task struct, public API declarations, TASK_* error codes
- `dpmi.h`: DPMI_LockMemoryRegion(), DPMI_Lock(), etc. — protected-mode memory management
- `usrhooks.h` (optional): USRHOOKS_GetMem, USRHOOKS_FreeMem — custom allocator

**DOS/x86 Functions (defined elsewhere):**
- `_dos_getvect()`, `_dos_setvect()` — interrupt vector table access
- `_chain_intr()` — chain to previous interrupt handler
- `_enable()`, `_disable()` — global interrupt control (POPFD, PUSHFD/CLI)
- `int386()` — execute real-mode interrupt (DPMI)
- `outp()`, `inp()` — I/O port read/write (8253 timer control)
- `malloc()`, `free()` — standard C memory allocation
- `memset()` — memory initialization

**Watcom-specific:**
- `#pragma aux` — inline assembly function declarations (GetStack, SetStack, DPMI calls)

**Hardware:**
- 8253/8254 programmable interval timer at ports 0x40 (data) and 0x43 (control)
- INT 8 (Timer Tick interrupt)
- Base timer frequency: 1.192030 MHz

# audiolib/source/task_man.h
## File Purpose
Public header for a low-level timer task scheduler. Defines the interface for registering, managing, and dispatching time-based tasks with configurable execution rates and priorities. Designed for use in both regular code and interrupt contexts.

## Core Responsibilities
- Define task structure and error codes for task management
- Provide scheduling/termination API for time-based tasks
- Manage task dispatch and rate control
- Track interrupt execution context for code that operates in both environments
- Provide memory locking primitives for interrupt-safe critical sections

## External Dependencies
- **No includes visible** in this header (pure interface definition)
- Clients must include this header to use the task scheduling API
- Task service callbacks receive task structure pointer for access to registered data

# audiolib/source/user.c
## File Purpose
Provides command-line argument parsing utilities for detecting parameters and retrieving associated values. Supports DOS-style parameter syntax (`-param` or `/param`). Intended for engine initialization and configuration from the command line.

## Core Responsibilities
- Parse command-line arguments (`_argc` / `_argv`)
- Check for presence of flags prefixed with `-` or `/`
- Retrieve text values following parameter flags
- Case-insensitive parameter matching using `stricmp`

## External Dependencies
- **Standard includes:** `<dos.h>`, `<string.h>`
- **External globals:** `_argc`, `_argv` (C runtime)
- **Defined elsewhere:** `stricmp()` (case-insensitive string comparison; likely from libc or DOS libraries)

# audiolib/source/user.h
## File Purpose
Public header file for the USER module, declaring the interface for parameter checking and text retrieval functions. Part of the Apogee audio library's user configuration system.

## Core Responsibilities
- Declare function to validate/check user-provided parameters
- Declare function to retrieve text values associated with parameters
- Provide public API for parameter and configuration handling

## External Dependencies
- Standard C library (implied by function signatures)
- No explicit includes visible in this header

# audiolib/source/usrhooks.c
## File Purpose
Provides wrapper functions for memory allocation and deallocation, allowing the calling program to customize or intercept dynamic memory operations in the audio library. The module returns standardized error codes rather than raw malloc/free semantics.

## Core Responsibilities
- Wrap malloc/free operations with consistent error reporting
- Abstract memory management behind a defined interface
- Allow callers to override or monitor memory allocation behavior
- Validate allocation requests and return status codes

## External Dependencies
- `stdlib.h` — `malloc()`, `free()`
- `usrhooks.h` — `USRHOOKS_Errors` enum (defines return codes: `USRHOOKS_Ok`, `USRHOOKS_Error`, `USRHOOKS_Warning`)

# audiolib/source/usrhooks.h
## File Purpose
Public header defining memory allocation hooks for the audio library. Allows the calling program to control memory management operations that the audio library performs, enabling custom allocation strategies or restricted environments.

## Core Responsibilities
- Define error codes for hook function operations
- Declare function prototypes for memory allocation/deallocation
- Establish a standard interface between audio library and caller for memory operations
- Enable the calling program to intercept and customize memory management

## External Dependencies
None.

