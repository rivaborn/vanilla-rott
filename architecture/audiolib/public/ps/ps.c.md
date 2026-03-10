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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `fx_device` | struct | Device capability query result; holds max voices, sample bits, and channels (from fx_man.h) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SoundCardNames` | char *[] | file-static | Array of 8 human-readable sound card names for display |
| `SoundCardNums` | int[] | file-static | Array of 8 sound card type constants corresponding to names |

## Key Functions / Methods

### main
- **Signature:** `void main(int argc, char *argv[])`
- **Purpose:** Entry point. Parses arguments, loads audio file, initializes sound card, runs playback loop, and cleans up.
- **Inputs:** Command-line arguments (filename + optional CARD, VOICES, BITS, RATE, REVERB, MONO/STEREO flags).
- **Outputs/Return:** Exit status (0 on success, 1 on error).
- **Side effects:** Initializes sound card hardware, allocates heap memory for audio buffer, prints to stdout, reads from disk, calls FX_Init/FX_Shutdown.
- **Calls:** GetUserText, CheckUserParm, LoadFile, DefaultExtension, FX_SetupCard, FX_Init, FX_SetReverb, FX_SetVolume, FX_PlayWAV, FX_PlayVOC, FX_PlayRaw, FX_StopAllSounds, FX_Shutdown, FX_ErrorString, stricmp, getch, free, exit.
- **Notes:** Loops on user input until ESC (ASCII 27) is pressed. Tries loading with .wav and .voc extensions if filename has no extension. Defaults: card=0 (Sound Blaster), voices=4, bits=8, channels=1 (mono), rate=11000 Hz, reverb=0.

### LoadFile
- **Signature:** `char *LoadFile(char *filename, int *length)`
- **Purpose:** Load an audio file into dynamically allocated memory.
- **Inputs:** Filename string, pointer to output length variable.
- **Outputs/Return:** Pointer to allocated audio buffer; sets *length to file size in bytes; returns NULL on open failure.
- **Side effects:** Allocates heap memory via malloc; opens and reads file; calls exit(1) on malloc or fread failure.
- **Calls:** fopen, fseek, ftell, malloc, fread, fclose, printf, exit.
- **Notes:** On malloc or fread failure, prints error and terminates immediately rather than returning error code.

### GetUserText
- **Signature:** `char *GetUserText(const char *parameter)`
- **Purpose:** Extract the value of a command-line parameter in the form `PARAM=value`.
- **Inputs:** Parameter name (e.g., "VOICES", "CARD").
- **Outputs/Return:** Pointer to the value string (text after '='), or NULL if not found.
- **Side effects:** Accesses extern _argc and _argv.
- **Calls:** strlen, strnicmp.
- **Notes:** Case-insensitive search; stops at first match; parameter must be followed by '=' sign.

### CheckUserParm
- **Signature:** `int CheckUserParm(const char *parameter)`
- **Purpose:** Check if a flag parameter (preceded by '-' or '/') exists in the command line.
- **Inputs:** Parameter name (e.g., "?").
- **Outputs/Return:** TRUE if found, FALSE otherwise.
- **Side effects:** Accesses extern _argc and _argv.
- **Calls:** strlen, stricmp.
- **Notes:** Only checks arguments that start with '-' or '/' prefix.

### DefaultExtension
- **Signature:** `void DefaultExtension(char *path, char *extension)`
- **Purpose:** Append a file extension if the path does not already have one.
- **Inputs:** Path string, extension string (should include dot, e.g., ".wav").
- **Outputs/Return:** None; modifies path in-place.
- **Side effects:** Modifies the input path buffer.
- **Calls:** strlen, strcat.
- **Notes:** Walks backwards from path end until '\' or path start is found; if '.' is encountered, assumes an extension exists and returns early.

## Control Flow Notes

**Initialization phase (main):** Parses arguments → loads audio file → calls FX_SetupCard and FX_Init to initialize sound card.

**Main loop (main):** Repeatedly waits for keypress via getch(); on each key (except ESC), plays the loaded audio file using the appropriate function (FX_PlayWAV, FX_PlayVOC, or FX_PlayRaw based on file extension).

**Shutdown (main):** On ESC, calls FX_StopAllSounds, frees the audio buffer, calls FX_Shutdown, and exits.

## External Dependencies

- **Includes:** `<conio.h>` (getch), `<dos.h>` (legacy DOS headers), `<stdlib.h>` (malloc, exit), `<stdio.h>` (fopen, fread, printf), `<string.h>` (strcpy, strcat, strlen, stricmp, strnicmp), `"fx_man.h"` (audio library API).
- **External symbols (defined elsewhere):** FX_SetupCard, FX_Init, FX_SetReverb, FX_SetVolume, FX_PlayWAV, FX_PlayVOC, FX_PlayRaw, FX_StopAllSounds, FX_Shutdown, FX_ErrorString, SoundBlaster, Awe32, ProAudioSpectrum, SoundMan16, SoundScape, UltraSound, SoundSource, TandySoundSource (sound card type constants from fx_man.h or sndcards.h).
- **Extern globals:** _argc, _argv (DOS runtime command-line storage).
