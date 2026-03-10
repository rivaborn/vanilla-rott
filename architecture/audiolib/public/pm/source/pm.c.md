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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `songposition` | struct | Playback position tracking with tickposition, milliseconds, measure, beat, tick fields |
| `union REGS` | union | DOS BIOS interrupt register container for cursor control (ax, cx, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SoundCardNames[]` | char* array | static | Names of 10 supported sound cards (GenMidi, SoundCanvas, Awe32, etc.) |
| `SoundCardNums[]` | int array | static | Parallel array of sound card enum constants matching SoundCardNames |

## Key Functions / Methods

### main
- **Signature:** `void main(int argc, char *argv[])`
- **Purpose:** Entry point. Parses arguments, initializes sound card, loads and plays MIDI with interactive control loop.
- **Inputs:** Command-line arguments (`CARD=N`, `MPU=addr`, `TIMBRE=file`, `POSITION=m:b:t`, `TIME=min:sec:ms`)
- **Outputs/Return:** Program exit via `exit()` calls
- **Side effects:** Allocates heap memory for MIDI/timbre buffers, initializes MUSIC subsystem, outputs to stdout, performs DOS BIOS interrupt calls for cursor control, reads keyboard input
- **Calls:** `CheckUserParm()`, `GetUserText()`, `DefaultExtension()`, `LoadTimbres()`, `LoadMidi()`, `MUSIC_Init()`, `MUSIC_PlaySong()`, `MUSIC_SetSongPosition()`, `MUSIC_SetSongTime()`, `MUSIC_GetSongPosition()`, `MUSIC_GetSongLength()`, `MUSIC_StopSong()`, `MUSIC_Shutdown()`, `TurnOffTextCursor()`, `TurnOnTextCursor()`, `free()`, `kbhit()`, `getch()`
- **Notes:** Interactive loop polls `kbhit()` and processes F/f (forward measure), R/r (rewind), G/g (goto position), ESC (exit); two position modes (measure:beat:tick vs time-based); hardcoded defaults (GenMidi, address 0x330)

### LoadTimbres
- **Signature:** `void LoadTimbres(char *timbrefile)`
- **Purpose:** Load instrument definitions (timbre bank) from file into memory and register with music subsystem.
- **Inputs:** `timbrefile` – path to timbre file
- **Outputs/Return:** None (void)
- **Side effects:** Allocates heap memory, file I/O, calls `MUSIC_RegisterTimbreBank()`; exits on error
- **Calls:** `fopen()`, `fseek()`, `ftell()`, `malloc()`, `fread()`, `fclose()`, `MUSIC_RegisterTimbreBank()`, `exit()`
- **Notes:** Exits program on file open, memory allocation, or read failures; no validation of file format

### LoadMidi
- **Signature:** `char *LoadMidi(char *filename)`
- **Purpose:** Load complete MIDI file into heap-allocated memory buffer.
- **Inputs:** `filename` – path to MIDI file
- **Outputs/Return:** Pointer to allocated buffer containing file contents
- **Side effects:** Allocates heap memory, file I/O; exits on error
- **Calls:** `fopen()`, `fseek()`, `ftell()`, `malloc()`, `fread()`, `fclose()`, `exit()`
- **Notes:** Exits program on failures; caller must `free()` returned pointer; no MIDI validation

### GetUserText
- **Signature:** `char *GetUserText(const char *parameter)`
- **Purpose:** Extract argument value from command line for parameters in "PARAM=value" format.
- **Inputs:** `parameter` – parameter name to search for
- **Outputs/Return:** Pointer to value string after '=' if found; NULL otherwise
- **Side effects:** None (read-only access to _argc/_argv)
- **Calls:** `strlen()`, `strnicmp()`
- **Notes:** Case-insensitive; scans _argc/_argv extern globals (DOS runtime); starts from argv[1]

### CheckUserParm
- **Signature:** `int CheckUserParm(const char *parameter)`
- **Purpose:** Check if a command-line flag exists (preceded by '-' or '/').
- **Inputs:** `parameter` – parameter name to search for
- **Outputs/Return:** TRUE if found, FALSE otherwise
- **Side effects:** None
- **Calls:** `stricmp()`
- **Notes:** Scans _argc/_argv extern globals; case-insensitive; only matches flags with '-' or '/' prefix

### DefaultExtension
- **Signature:** `void DefaultExtension(char *path, char *extension)`
- **Purpose:** Append file extension if filename does not already have one.
- **Inputs:** `path` – file path (modified in-place); `extension` – extension string including '.'
- **Outputs/Return:** None (modifies path in-place)
- **Side effects:** String mutation via `strcat()`
- **Calls:** `strlen()`, `strcat()`
- **Notes:** Searches backward from end of path for '.' or path separator ('\\'); assumes sufficient buffer space

### TurnOffTextCursor / TurnOnTextCursor
- **Signature:** `void TurnOffTextCursor(void)` / `void TurnOnTextCursor(void)`
- **Purpose:** Disable/enable DOS text-mode cursor via BIOS INT 0x10 call.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** BIOS interrupt (INT 0x10, AH=01h); modifies hardware cursor state
- **Calls:** `int386()` or `int86()` depending on `__FLAT__` (32-bit vs 16-bit)
- **Notes:** Conditional compilation for protected/real mode; hardcoded register values (0x0100 for AH/AL, 0x2000 for off, 0x0708 for on)

## Control Flow Notes
**Init phase:** `main()` parses environment/CLI args → initializes MUSIC subsystem → loads MIDI and optionally timbre files → sets volume/playback position if specified.

**Main loop:** Continuous `while(ch != 27)` (until ESC) polls `MUSIC_GetSongPosition()`, renders position to console, and processes single-key commands (F/R/G) to adjust playback position.

**Shutdown:** Stops song, frees MIDI buffer, calls `MUSIC_Shutdown()`.

## External Dependencies
- **Local headers:** `music.h` (MUSIC_Init, MUSIC_PlaySong, MUSIC_SetSongPosition, MUSIC_GetSongPosition, MUSIC_GetSongLength, MUSIC_StopSong, MUSIC_Shutdown, MUSIC_ErrorString, MUSIC_SetVolume, MUSIC_RegisterTimbreBank); `sndcards.h` (GenMidi, SoundCanvas, Awe32, WaveBlaster, SoundBlaster, ProAudioSpectrum, SoundMan16, Adlib, SoundScape, UltraSound)
- **Standard C:** stdio, stdlib, string (printf, exit, malloc, free, strcpy, strcat, strlen, strnicmp, stricmp, fopen, fread, fclose, fseek, ftell, sscanf)
- **DOS/Platform-specific:** conio (kbhit, getch), dos (union REGS, int86, int386, SEEK_END, SEEK_SET)
