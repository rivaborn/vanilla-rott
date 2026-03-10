# audiolib/public/ps/ps.c — Enhanced Analysis

## Architectural Role

This is a standalone command-line developer utility for testing sound playback across multiple sound card types supported by the FX audio abstraction layer. It sits atop the audio subsystem (fx_man.h) which abstracts hardware differences (Sound Blaster, AWE32, GUS, etc.). The utility is not part of the game engine proper but a testing/validation tool in the audiolib development suite, paired with similar utilities like the MIDI player (pm.c) in the same audiolib/public directory.

## Key Cross-References

### Incoming (who depends on this file)
- **None in game engine**: This is a standalone utility, not linked into the main codebase
- **Shared utility**: `CheckUserParm` function is reused by `audiolib/public/pm/source/pm.c` (likely a MIDI player utility with identical command-line parsing needs)

### Outgoing (what this file depends on)
- **FX subsystem** (fx_man.h): FX_SetupCard, FX_Init, FX_PlayWAV, FX_PlayVOC, FX_PlayRaw, FX_SetReverb, FX_SetVolume, FX_StopAllSounds, FX_Shutdown, FX_ErrorString
- **Standard C runtime**: malloc/free, fopen/fread/fclose, getch, strlen, strcpy, sscanf
- **DOS runtime**: extern _argc and _argv for command-line argument access (not standard C)

## Design Patterns & Rationale

**Simple imperative/procedural flow**: No state machines, no callbacks, no async patterns. The program is synchronous: initialize → loop-on-input → shutdown. This reflects 1990s single-threaded DOS/early-Windows conventions.

**Abstraction via FX manager**: Audio card differences (Sound Blaster vs. AWE32 vs. GUS) are abstracted by fx_man.h. The utility doesn't know or care about card-specific code; it talks only to the FX API. This insulates developers from hardware details.

**Format-agnostic playback**: Rather than requiring users to specify file format, the utility tries .wav first, then .voc, then raw. This reduces command-line friction but lacks explicit validation (relies on format handlers to reject bad data).

**Parameter parsing by convention**: No getopt or formal arg parser. Instead, `GetUserText` and `CheckUserParm` scan argv manually looking for KEY=VALUE and -FLAG patterns. This was common in 1990s tools before standard library support.

## Data Flow Through This File

1. **Initialization**: Command-line parsed → defaults applied (card=0, voices=4, bits=8, rate=11000, mono, no reverb) → user values override via GetUserText/CheckUserParm
2. **File loading**: Filename + fallback extensions → LoadFile allocates buffer from disk → single buffer held in memory for duration
3. **Hardware init**: FX_SetupCard queries device capabilities; FX_Init sets up mixing engine with user parameters
4. **Playback loop**: getch blocks for user input; each keypress (except ESC=27) triggers FX_PlayWAV/VOC/Raw with the same buffer; voice handle is returned but only partially validated (checks < FX_Ok but uses wrong variable on error)
5. **Cleanup**: FX_StopAllSounds, free buffer, FX_Shutdown, exit

## Learning Notes

**Era-specific idioms**: This code exemplifies 1990s DOS/early-Windows C:
- `void main()` (not int) and direct exit() calls rather than proper return
- Extern _argc/_argv instead of standard function parameters
- getch() for synchronous input (blocking, not event-driven)
- No error return codes; instead printf + exit
- Hardcoded buffer sizes (filename[128]) and string ops without bounds checking

**Modern contrast**: A 2020s version would use:
- Proper error enums or Result<T, E> types instead of exit()
- Async event loops or callback-based input instead of blocking getch()
- Dynamic string allocation or fixed-size checks to prevent overflow
- Standard argument parsing library (getopt, clap, etc.)

**Relationship to game engine**: Unlike the game's core rendering/actor/AI systems, this is a thin command-line wrapper over the audio abstraction. It validates that the FX system works correctly for a given sound card and audio file. Developers would run it to verify hardware support before shipping.

## Potential Issues

1. **Wrong variable in error check** (line ~285): `if ( voice < FX_Ok ) { printf( "Sound error - %s\n", FX_ErrorString( status ) ); }` — should be `FX_ErrorString( voice )`, not `status` (which is never updated in the loop).
2. **Incomplete null validation**: LoadFile returns NULL on fopen failure but doesn't validate it in all fallback branches (lines ~195-209 do check, but error message repeats "argv[1]" without distinguishing .wav vs. .voc vs. raw attempts).
3. **exit() in helper function**: LoadFile calls exit(1) on malloc/fread failure (lines ~256-260), violating separation of concerns. Caller should decide whether to exit or retry.
4. **No file format validation**: Relies entirely on extension (line ~283 uses stricmp on last 3 chars). A file named "sound.wav" that is actually VOC data will be mishandled.
5. **Buffer overflow risk** (minor in practice): `strcpy( filename, argv[1] )` and `DefaultExtension` can overflow filename[128] if argv[1] is > 127 chars or combined with extension > 128.
6. **No reverb validation**: Reverb parameter is read but never validated; FX_SetReverb silently accepts any int.
