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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `midifuncs` | struct | Function pointer table for device-specific MIDI operations (NoteOff, NoteOn, ControlChange, ProgramChange, etc.) |
| `songposition` | struct | Song position tracking (tickposition, milliseconds, measure, beat, tick) |
| `task` | struct (from task_man.h) | Lightweight task descriptor used for fade scheduling |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MUSIC_SoundDevice` | int | global | Currently selected sound device ID (enum from sndcards.h) |
| `MUSIC_ErrorCode` | int | global | Current error code for error reporting |
| `MUSIC_MidiFunctions` | midifuncs | static | Function pointers populated at init time by device-specific init functions |
| `MUSIC_FadeTask` | task* | static | Active fade effect task pointer; NULL if no fade in progress |
| `MUSIC_FadeLength` | int | static | Remaining fade time in 25ms units |
| `MUSIC_FadeRate` | int | static | Volume change rate (fixed-point: shifted left 7 bits) |
| `MUSIC_CurrentFadeVolume` | unsigned | static | Current fade volume (fixed-point) |
| `MUSIC_LastFadeVolume` | unsigned | static | Last reported fade volume to avoid redundant updates |
| `MUSIC_EndingFadeVolume` | int | static | Target volume for fade operation |

## Key Functions / Methods

### MUSIC_Init
- **Signature:** `int MUSIC_Init(int SoundCard, int Address)`
- **Purpose:** Initialize the music system by selecting a sound device and configuring its MIDI driver
- **Inputs:** `SoundCard` (enum soundcardnames), `Address` (I/O port for MPU-401 devices)
- **Outputs/Return:** `MUSIC_Ok` on success, `MUSIC_Error` on failure
- **Side effects:** Locks memory via LL_LockMemory(), populates MUSIC_MidiFunctions, sets MUSIC_SoundDevice, initializes MIDI patch map
- **Calls:** LL_LockMemory(), MUSIC_InitFM(), MUSIC_InitMidi(), MUSIC_InitAWE32(), MUSIC_InitGUS()
- **Notes:** Dispatches to card-specific initializers based on SoundCard enum; unlocks memory on error

### MUSIC_Shutdown
- **Signature:** `int MUSIC_Shutdown(void)`
- **Purpose:** Terminate music playback and release hardware resources
- **Inputs:** None
- **Outputs/Return:** `MUSIC_Ok`
- **Side effects:** Stops any active song and fade, resets device-specific hardware, unlocks memory
- **Calls:** MIDI_StopSong(), MUSIC_StopFade(), device-specific shutdown functions (AL_Shutdown, MPU_Reset, AWE32_Shutdown, GUSMIDI_Shutdown, etc.), LL_UnlockMemory()
- **Notes:** Handles device-specific cleanup (e.g., BLASTER_RestoreMidiVolume for cards with mixer)

### MUSIC_PlaySong
- **Signature:** `int MUSIC_PlaySong(unsigned char *song, int loopflag)`
- **Purpose:** Begin MIDI song playback
- **Inputs:** `song` (MIDI file buffer), `loopflag` (MUSIC_LoopSong or MUSIC_PlayOnce)
- **Outputs/Return:** `MUSIC_Ok` on success, `MUSIC_Warning` if invalid device or MIDI error
- **Side effects:** Stops current song before starting new one; sets MUSIC_ErrorCode on failure
- **Calls:** MIDI_StopSong(), MIDI_PlaySong()
- **Notes:** Valid device must be initialized; routes to MIDI layer for actual playback

### MUSIC_SetVolume
- **Signature:** `void MUSIC_SetVolume(int volume)`
- **Purpose:** Set overall music volume
- **Inputs:** `volume` (0–255, clamped to valid range)
- **Outputs/Return:** None
- **Side effects:** Calls device-specific SetVolume if device is initialized
- **Calls:** MIDI_SetVolume()
- **Notes:** No-op if no device initialized (MUSIC_SoundDevice == -1)

### MUSIC_FadeVolume
- **Signature:** `int MUSIC_FadeVolume(int tovolume, int milliseconds)`
- **Purpose:** Smoothly fade music volume over time using a background task
- **Inputs:** `tovolume` (target volume 0–255), `milliseconds` (fade duration)
- **Outputs/Return:** `MUSIC_Ok` on success, `MUSIC_Warning` if task scheduling failed
- **Side effects:** Schedules MUSIC_FadeRoutine via TS_ScheduleTask at 40Hz; stops any active fade first; sets global fade state variables
- **Calls:** MUSIC_StopFade(), TS_ScheduleTask(), TS_Dispatch()
- **Notes:** Some devices (PAS, GenMidi, SoundScape, SoundCanvas) use immediate volume set instead of gradual fade; uses fixed-point math (shift by 7 bits) for smooth interpolation

### MUSIC_FadeRoutine
- **Signature:** `static void MUSIC_FadeRoutine(task *Task)`
- **Purpose:** Executed by task scheduler (~25Hz); performs incremental volume fade
- **Inputs:** `Task` (scheduler context)
- **Outputs/Return:** None
- **Side effects:** Decrements MUSIC_FadeLength, updates MUSIC_CurrentFadeVolume, calls MIDI_SetVolume() if volume changed, terminates task when fade complete
- **Calls:** MIDI_SetVolume(), TS_Terminate()
- **Notes:** Uses hysteresis (MUSIC_LastFadeVolume) to avoid redundant volume updates; terminates itself when MUSIC_FadeLength reaches 0

### MUSIC_InitFM, MUSIC_InitMidi, MUSIC_InitAWE32, MUSIC_InitGUS
- **Signature:** `int MUSIC_Init<Device>(... midifuncs *Funcs, ...)`
- **Purpose:** Device-specific initialization; populates Funcs with device driver function pointers
- **Side effects:** Detects hardware, configures device, sets up mixer volume control if available
- **Calls:** Device-specific init/detection/setup functions (AL_Init, AL_DetectFM, BLASTER_*, MPU_*, etc.)
- **Notes:** All return MUSIC_Ok or MUSIC_Error; call MIDI_SetMidiFuncs(Funcs) at end to activate driver

## Control Flow Notes
1. **Initialization phase:** MUSIC_Init() detects sound device and calls appropriate MUSIC_Init<Device>() to populate MUSIC_MidiFunctions
2. **Playback phase:** MUSIC_PlaySong() routes to MIDI_PlaySong(), which invokes function pointers from MUSIC_MidiFunctions
3. **Volume control:** MUSIC_SetVolume() directly updates volume; MUSIC_FadeVolume() schedules MUSIC_FadeRoutine as a background task that runs every 40ms
4. **Shutdown phase:** MUSIC_Shutdown() stops playback, terminates any active fade task, calls device-specific cleanup

## External Dependencies
- **task_man.h:** Task scheduling (TS_ScheduleTask, TS_Terminate, TS_Dispatch)
- **sndcards.h:** Sound card enum definitions
- **midi.h:** MIDI playback layer (MIDI_PlaySong, MIDI_SetVolume, MIDI_SetMidiFuncs, etc.)
- **al_midi.h, pas16.h, blaster.h, gusmidi.h, mpu401.h, awe32.h, sndscape.h:** Device-specific drivers (defined elsewhere)
- **ll_man.h:** Memory locking (LL_LockMemory, LL_UnlockMemory)
- **user.h:** Command-line parameter checking (USER_CheckParameter)
