# Subsystem Overview

## Purpose
audiolib/public is Jim Dosé's standalone audio library, providing a unified interface for sound card initialization, sound effects playback, MIDI music control, and interrupt-driven task scheduling. It abstracts hardware differences across multiple sound card types (Sound Blaster, Adlib, UltraSound, Pro Audio Spectrum, etc.) and delivers high-level audio functionality for the Rise of the Triad engine, supporting effects playback (VOC, WAV, raw PCM) with 3D positioning and MIDI music playback with seeking and volume control.

## Key Files
| File | Role |
|------|------|
| audiolib/public/include/fx_man.h | Sound effects API—sound card detection, voice allocation, sample playback with pitch/panning/volume, 3D positioning, reverb |
| audiolib/public/include/music.h | MIDI music API—song playback, seek/pause/stop, volume/fade control, timbre bank configuration, channel routing |
| audiolib/public/include/sndcards.h | Sound card type enumeration and library version identifier |
| audiolib/public/include/task_man.h | Timer-based task scheduler API—task creation, priority management, interrupt-safe dispatch |
| audiolib/public/include/usrhooks.h | Memory hook interface—allows calling programs to customize malloc/free behavior |
| audiolib/public/pm/source/pm.c | MIDI player utility—reference implementation of music.h API |
| audiolib/public/ps/source/ps.c | Sound effects player utility—reference implementation of fx_man.h API |
| audiolib/public/timer/source/timer.c | Task scheduler demo—reference implementation of task_man.h API |

## Core Responsibilities
- Detect, initialize, and configure sound cards and MIDI devices, abstracting hardware-specific I/O and interrupt handling
- Manage voice and MIDI channel allocation with priority-based queuing for competing sounds
- Play audio in multiple formats (WAV, VOC, raw PCM) with real-time pitch, panning, volume, and 3D spatial effects
- Support looped/non-looped playback with completion callbacks
- Control MIDI music (play, pause, seek by time/ticks/measure, volume fade, per-channel routing)
- Provide interrupt-safe task scheduling for DOS timer-driven periodic execution
- Intercept and abstract memory allocation via usrhooks, enabling custom memory policies and alignment enforcement

## Key Interfaces & Data Flow
**Exposes to game code:**
- FX_MAN: Sound card setup, sample playback with effects (pitch/pan/3D/reverb), voice management
- MUSIC: Song loading and playback, seeking, volume control, timbre bank selection, channel mapping
- TASK_MAN: Task scheduling with dynamic rate adjustment and priority dispatch
- USRHOOKS: Customizable malloc/free hooks for application-controlled memory management

**Consumes from:**
- Calling code (game engine): Sample/song file data, user input for playback control, custom memory allocators
- Hardware: Sound card I/O registers, DMA, and interrupt vectors (accessed via abstracted drivers)
- DOS environment: System timer interrupts, command-line arguments, runtime C library (stdlib, conio, dos.h)

## Runtime Role
- **Initialization:** FX_Init or MUSIC_Init during game startup detects and configures sound card; TASK_MAN hooks system timer
- **Frame execution:** Audio playback proceeds asynchronously; TASK_MAN dispatcher invoked (interrupt-driven or polled) to execute scheduled tasks at specified rates
- **Shutdown:** FX_Shutdown, MUSIC_Shutdown, and TASK_MAN cleanup release hardware resources and memory on game exit

## Notable Implementation Details
- Sound card abstraction via sndcards.h enum; hardware-specific drivers (interrupt handlers, DMA control, I/O sequencing) remain encapsulated in implementation files not visible in headers
- USRHOOKS enforces dword-aligned pointer returns and standardized error codes (Ok/Error) to enable memory pooling or tracking by calling code
- MIDI timbre banks support per-channel callback routing and custom controller mapping for flexible music control
- 3D audio uses angle and distance parameters to apply spatial effects (panning and distance attenuation) to mono/stereo channels
- Task scheduler designed for DOS interrupt context with memory locking primitives to ensure interrupt-safe execution
- Reference implementations (pm.c, ps.c, timer.c) demonstrate API usage patterns for sound effects, MIDI playback, and task scheduling respectively
