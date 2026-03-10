# audiolib/public/pm/source/pm.c — Enhanced Analysis

## Architectural Role
PM.C is a **standalone command-line MIDI player utility** for DOS, not integrated into the main game engine. It demonstrates the **public MUSIC API** (`MUSIC_*` subsystem) that abstracts away the low-level sound card drivers (ADLIBFX, AWE32, AL_MIDI, BLASTER, etc.). This utility serves as both a reference implementation and user-facing tool for testing music playback across multiple sound card configurations.

## Key Cross-References

### Incoming (who depends on this file)
- **None detected.** PM.C defines only a `main()` entry point; no other files call its functions. It's a standalone executable tool, not a library component.

### Outgoing (what this file depends on)
- **MUSIC subsystem** (`music.h`): Calls `MUSIC_Init()`, `MUSIC_PlaySong()`, `MUSIC_SetSongPosition()`, `MUSIC_SetSongTime()`, `MUSIC_GetSongPosition()`, `MUSIC_GetSongLength()`, `MUSIC_StopSong()`, `MUSIC_Shutdown()`, `MUSIC_SetVolume()`, `MUSIC_RegisterTimbreBank()`, `MUSIC_ErrorString()`
  - This is the *public, high-level* audio API. The MUSIC module abstracts sound card selection and driver management.
- **Sound card enums** (`sndcards.h`): Defines `GenMidi`, `SoundCanvas`, `Awe32`, `WaveBlaster`, `SoundBlaster`, `ProAudioSpectrum`, `SoundMan16`, `Adlib`, `SoundScape`, `UltraSound`
  - These constants map directly to the `SoundCardNums[]` array in pm.c, enabling pluggable card selection.
- **DOS/platform APIs**: `conio.h` (keyboard I/O), `dos.h` (interrupt calls), `stdio.h`/`stdlib.h` (file/memory management)

## Design Patterns & Rationale

1. **Parallel Lookup Arrays**  
   - `SoundCardNames[]` (strings) and `SoundCardNums[]` (enum constants) are kept in sync  
   - **Rationale:** Enables human-readable CLI selection (`CARD=2` → "Awe 32") without hardcoding strings in the MUSIC subsystem  
   - **Trade-off:** Manual synchronization burden; could break if arrays misalign

2. **Direct File Streaming to Memory**  
   - MIDI and timbre files are loaded entirely into heap buffers, passed as opaque pointers to the MUSIC subsystem  
   - **Rationale:** DOS-era simplicity; avoids I/O during playback. The MUSIC subsystem then parses/interprets these buffers  
   - **Trade-off:** No streaming, high memory use for large files; no in-place file validation

3. **Two-Mode Position Seeking**  
   - Users can specify playback start via measure:beat:tick (musical) OR time (minutes:seconds:ms)  
   - **Rationale:** Caters to both score-aware composition workflows and generic media players  
   - **Implementation:** `gotopos` flag selects which API (`MUSIC_SetSongPosition()` vs `MUSIC_SetSongTime()`) is called

4. **Interrupt-Based Cursor Control**  
   - `TurnOffTextCursor()` / `TurnOnTextCursor()` use raw DOS INT 0x10 calls instead of portability layers  
   - **Rationale:** Direct hardware control for minimal overhead; typical of early-90s DOS game tools  
   - **Trade-off:** Non-portable; assumes text-mode cursor architecture (INT 0x10 AH=01h)

5. **Command-Line Parsing via Environment & Args**  
   - Supports both environment variable (`PM=card,address`) and argv parsing (`CARD=N`, `MPU=addr`)  
   - **Rationale:** Flexibility for batch automation; argv takes precedence (explicit > implicit)

## Data Flow Through This File

```
CLI Args / Env Variables
    ↓
GetUserText() / CheckUserParm()  [parse CARD=, MPU=, TIMBRE=, POSITION=, TIME=]
    ↓
MUSIC_Init(card, address)        [initialize sound subsystem with selected hardware]
    ↓
LoadTimbres(timbrefile)          [optional] → MUSIC_RegisterTimbreBank()
    ↓
LoadMidi(filename)               → malloc() MIDI buffer → returned to main
    ↓
MUSIC_PlaySong(MidiPtr, loop)    [start playback]
    ↓
Main Loop:
  • MUSIC_GetSongPosition()      [query playback state]
  • Keyboard input (kbhit/getch) → F/R/G/ESC commands
  • MUSIC_SetSongPosition() / MUSIC_SetSongTime()  [seek if user requested]
    ↓
ESC key → MUSIC_StopSong() → free(MidiPtr) → MUSIC_Shutdown()
```

## Learning Notes

1. **Public API Abstraction**: PM.C never touches the low-level drivers (ADLIBFX, AWE32, BLASTER, etc.). The MUSIC subsystem is the *only* entry point, demonstrating clean API boundaries in audio engine design. This is idiomatic for mid-90s audio libraries where driver complexity needed hiding.

2. **DOS-Specific Idioms**:
   - `strnicmp()` (case-insensitive) for CLI parsing reflects DOS command-line conventions
   - Direct INT 0x10 calls rather than abstracted cursor APIs
   - `_argc` / `_argv` extern globals (Watcom/Borland DOS convention) instead of function parameters
   - No error recovery; immediate `exit()` on file/memory failures

3. **Musical Position Semantics**: The `songposition` struct (measure/beat/tick, milliseconds) reveals the engine's dual awareness of *musical time* (for score-based sequencing) and *wall-clock time* (for synchronization). Modern engines typically unify these; ROTT keeps them separate.

4. **Polling-Based Interactivity**: The main loop is a simple `while(kbhit())` poll, not event-driven. Typical of real-time DOS tools with tight frame-rate synchronization elsewhere.

5. **No Streaming or Adaptive Loading**: Files are fully buffered before playback. Contrast with modern engines that stream large audio/MIDI and use adaptive buffering.

## Potential Issues

1. **Buffer Overflow Risk in `DefaultExtension()`**: Calls `strcat(path, extension)` without bounds checking. If `path` buffer is undersized, this will overflow.
   - **Severity**: High (DOS crash/stack corruption)
   - **Mitigation**: Caller must ensure `filename[128]` is sufficient

2. **Undefined Behavior on File Parse Errors**: `sscanf()` calls assume well-formed input (e.g., `POSITION=m:b:t`). Malformed input silently leaves variables uninitialized or partially set.
   - **Example**: `POSITION=5:x:10` parses `5` into `measure`, leaves `beat`/`tick` uninitialized

3. **Memory Leak on Early Exit**: If `MUSIC_Init()` fails, the program exits without freeing `SongPtr` (which is NULL anyway, but pattern is fragile). If `LoadMidi()` happens before `MUSIC_Init()` and init fails, the MIDI buffer leaks.
   - **Current mitigation**: MIDI loading is *after* MUSIC_Init succeeds, so leaks don't occur in this version
   - **Fragility**: Code order is implicit contract

4. **Sound Card Index Out-of-Bounds**: The check `if ( ( card < 0 ) || ( card >= NUMCARDS ) )` prints an error but *does not exit* — execution continues with an invalid array index to `SoundCardNums[card]`.
   - **Severity**: High (use-of-uninitialized-memory to MUSIC_Init)
