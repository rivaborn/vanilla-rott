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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `VOICE` | struct | FM voice state: key, velocity, channel, timbre, pitch, status, port |
| `VOICELIST` | struct | Linked list container for voices (start/end pointers) |
| `CHANNEL` | struct | MIDI channel state: active voices, timbre, pitch bend, volume, pan, detune |
| `TIMBRE` | struct | Instrument definition: envelope, level, SAVEK, wave, feedback, transpose, velocity (defined elsewhere) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `OctavePitch[]` | `unsigned[MAX_OCTAVE+1]` | static | Base pitch values per octave (0–7) |
| `NoteMod12[]` | `unsigned[MAX_NOTE+1]` | static | Precomputed note % 12 for pitch lookup |
| `NoteDiv12[]` | `unsigned[MAX_NOTE+1]` | static | Precomputed note / 12 for octave selection |
| `NotePitch[]` | `unsigned[FINETUNE_MAX+1][12]` | static | Fine-tuned pitch table (finetune × semitone) |
| `slotVoice[]` | `int[NUM_VOICES][2]` | static | Maps voice number to FM chip slot pairs (operator 0, 1) |
| `VoiceLevel[]` | `int[NumChipSlots][2]` | static | Amplitude lookup per slot and port |
| `VoiceKsl[]` | `int[NumChipSlots][2]` | static | Key scale level (attenuation) per slot and port |
| `offsetSlot[]` | `char[NumChipSlots]` | static | Register offset per chip slot |
| `VoiceReserved[]` | `int[NUM_VOICES*2]` | static | Bitmap of voices reserved by other drivers |
| `Voice[]` | `VOICE[NUM_VOICES*2]` | static | Voice state array (mono/stereo) |
| `Voice_Pool` | `VOICELIST` | static | Linked list of free voices |
| `Channel[]` | `CHANNEL[NUM_CHANNELS]` | static | MIDI channel states (16 channels) |
| `AL_LeftPort` | `int` | static | Left channel hardware port address (0x388 or 0x388) |
| `AL_RightPort` | `int` | static | Right channel hardware port address (0x388 or 0x38A) |
| `AL_Stereo` | `int` | static | Card hardware supports stereo (boolean) |
| `AL_SendStereo` | `int` | static | Currently sending stereo output (boolean) |
| `AL_OPL3` | `int` | static | Card is OPL3 mode (boolean) |
| `AL_MaxMidiChannel` | `int` | static | Maximum MIDI channel to process (default 16) |

## Key Functions / Methods

### AL_SendOutputToPort
- **Signature:** `void AL_SendOutputToPort(int port, int reg, int data)`
- **Purpose:** Low-level hardware write to Adlib register with timing delays.
- **Inputs:** Port (0x388/0x38A), register address, 8-bit data value
- **Outputs/Return:** None (void)
- **Side effects:** Writes to hardware I/O ports; timing-critical delays via `inp()` reads
- **Calls:** `outp()`, `inp()` (system I/O)
- **Notes:** 6-cycle delay before data write, 27-cycle delay after. Comment indicates original was 35 cycles. Timing is critical for Adlib hardware.

### AL_SendOutput
- **Signature:** `void AL_SendOutput(int voice, int reg, int data)`
- **Purpose:** Wrapper to send output to appropriate port(s) based on voice and stereo mode.
- **Inputs:** Voice number, register address, data value
- **Outputs/Return:** None
- **Side effects:** Calls `AL_SendOutputToPort()` once or twice
- **Calls:** `AL_SendOutputToPort()`
- **Notes:** Routes mono output to single port; stereo mode sends to both left and right ports.

### AL_SetVoiceTimbre
- **Signature:** `static void AL_SetVoiceTimbre(int voice)`
- **Purpose:** Programs the FM voice with instrument timbre (operator envelopes, waveforms, feedback).
- **Inputs:** Voice number
- **Outputs/Return:** None
- **Side effects:** Reads from `ADLIB_TimbreBank[]`, writes multiple hardware registers
- **Calls:** `AL_SendOutput()`, `AL_SendOutputToPort()`
- **Notes:** Skips reprogramming if timbre unchanged. Channel 9 (drums) uses key + 128 as patch index.

### AL_SetVoiceVolume
- **Signature:** `static void AL_SetVoiceVolume(int voice)`
- **Purpose:** Calculates and programs voice output level based on velocity, channel volume, pan, and timbre.
- **Inputs:** Voice number
- **Outputs/Return:** None
- **Side effects:** Writes amplitude registers for primary and (if additive) secondary operator
- **Calls:** `AL_SendOutput()`, `AL_SendOutputToPort()`
- **Notes:** Applies channel volume scaling, pan control (stereo), and checks feedback bit for additive mode.

### AL_SetVoicePitch
- **Signature:** `static void AL_SetVoicePitch(int voice)`
- **Purpose:** Programs voice pitch using note, detune, and channel pitch bend offset.
- **Inputs:** Voice number
- **Outputs/Return:** None
- **Side effects:** Writes frequency/octave registers; calculates right-channel detune for stereo
- **Calls:** `AL_SendOutput()`, `AL_SendOutputToPort()`
- **Notes:** Combines octave pitch, fine-tuned pitch, and applies channel transpose/key offset. Stereo mode applies extra detune to right channel.

### AL_AllocVoice
- **Signature:** `static int AL_AllocVoice(void)`
- **Purpose:** Retrieves a free voice from the voice pool.
- **Inputs:** None
- **Outputs/Return:** Voice number, or `AL_VoiceNotFound` if pool empty
- **Side effects:** Removes voice from `Voice_Pool` linked list
- **Calls:** `LL_Remove()`
- **Notes:** Simple pool allocation without voice stealing.

### AL_GetVoice
- **Signature:** `static int AL_GetVoice(int channel, int key)`
- **Purpose:** Locates the voice associated with a key on a specific MIDI channel.
- **Inputs:** Channel number, MIDI note key
- **Outputs/Return:** Voice number, or `AL_VoiceNotFound`
- **Side effects:** None (read-only traversal)
- **Calls:** None (linked list walk)
- **Notes:** Supports polyphony by key lookup within channel's active voices.

### AL_ResetVoices
- **Signature:** `static void AL_ResetVoices(void)`
- **Purpose:** Resets all voice and channel state to defaults, rebuilds free voice pool.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears and reinitializes `Voice[]`, `Channel[]`, `Voice_Pool`; respects reserved voices
- **Calls:** `LL_Remove()`, `LL_AddToTail()`
- **Notes:** Respects `VoiceReserved[]` bitmap. Uses OPL3 doubling if card supports and not stereo.

### AL_CalcPitchInfo
- **Signature:** `static void AL_CalcPitchInfo(void)`
- **Purpose:** Precalculates lookup tables for note-to-pitch conversion.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Populates `NoteMod12[]` and `NoteDiv12[]`
- **Calls:** None
- **Notes:** Commented-out finetune calculation suggests dynamic pitch table generation was considered but replaced by static table.

### AL_NoteOn
- **Signature:** `void AL_NoteOn(int channel, int key, int velocity)`
- **Purpose:** Plays a MIDI note on the specified channel.
- **Inputs:** MIDI channel (0–15), key (0–127), velocity (1–127)
- **Outputs/Return:** None
- **Side effects:** Allocates voice, adds to channel, programs timbre/volume/pitch
- **Calls:** `AL_AllocVoice()`, `AL_NoteOff()`, `AL_SetVoiceTimbre()`, `AL_SetVoiceVolume()`, `AL_SetVoicePitch()`, `LL_AddToTail()`
- **Notes:** Steals drum note (channel 9) if no voices available. Treats velocity=0 as note-off.

### AL_NoteOff
- **Signature:** `void AL_NoteOff(int channel, int key, int velocity)`
- **Purpose:** Stops a MIDI note on the specified channel.
- **Inputs:** MIDI channel, key, velocity (unused)
- **Outputs/Return:** None
- **Side effects:** Clears note-on bit, moves voice back to free pool
- **Calls:** `AL_GetVoice()`, `LL_Remove()`, `LL_AddToTail()`, `AL_SendOutput()`, `AL_SendOutputToPort()`
- **Notes:** Stereo mode sends note-off to both ports independently.

### AL_ControlChange
- **Signature:** `void AL_ControlChange(int channel, int type, int data)`
- **Purpose:** Handles MIDI controller messages (volume, pan, detune, RPN pitch bend).
- **Inputs:** MIDI channel, controller type, controller value
- **Outputs/Return:** None
- **Side effects:** Updates channel state; may call pitch bend or voice reprogramming
- **Calls:** `AL_SetChannelVolume()`, `AL_SetChannelPan()`, `AL_SetChannelDetune()`, `AL_AllNotesOff()`, `AL_ResetVoices()`
- **Notes:** Implements pitch bend RPN (Registered Parameter Number) with MSB/LSB split.

### AL_ProgramChange
- **Signature:** `void AL_ProgramChange(int channel, int patch)`
- **Purpose:** Selects instrument (timbre) for a MIDI channel.
- **Inputs:** MIDI channel, patch/program number (0–255)
- **Outputs/Return:** None
- **Side effects:** Updates `Channel[].Timbre`; voices repaint timbre on next note
- **Calls:** None (lazy repaint on note-on)
- **Notes:** Changes take effect on next note-on, not retroactively.

### AL_SetPitchBend
- **Signature:** `void AL_SetPitchBend(int channel, int lsb, int msb)`
- **Purpose:** Applies pitch bend to all voices on a channel.
- **Inputs:** MIDI channel, LSB and MSB of 14-bit pitch bend value (center=0x2000)
- **Outputs/Return:** None
- **Side effects:** Updates channel pitch offset and detune; reprograms all active voices
- **Calls:** `AL_SetVoicePitch()` for each voice
- **Notes:** Respects channel pitch bend range from RPN. Converts bend to semitone offset.

### AL_Init
- **Signature:** `int AL_Init(int soundcard)`
- **Purpose:** Initializes the Adlib driver: locks memory, detects card, configures ports.
- **Inputs:** Sound card type enum
- **Outputs/Return:** `AL_Ok` or `AL_Error`
- **Side effects:** DPMI locks multiple memory regions; configures `AL_OPL3`, `AL_LeftPort`, `AL_RightPort`; initializes pitch table and voice state
- **Calls:** `DPMI_LockMemoryRegion()`, `DPMI_Lock()`, `BLASTER_GetCardSettings()`, `BLASTER_GetEnv()`, `AL_CalcPitchInfo()`, `AL_Reset()`, `AL_ResetVoices()`
- **Notes:** Stereo initialization disabled via comment ("takes too long, causes mouse driver issues"). Probes Sound Blaster environment.

### AL_Shutdown
- **Signature:** `void AL_Shutdown(void)`
- **Purpose:** Gracefully shuts down the Adlib driver: disables stereo, resets card, unlocks memory.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `AL_StereoOff()`, `AL_ResetVoices()`, `AL_Reset()`; DPMI unlocks all locked regions
- **Calls:** `AL_StereoOff()`, `AL_ResetVoices()`, `AL_Reset()`, `DPMI_UnlockMemoryRegion()`, `DPMI_Unlock()` (22 times)
- **Notes:** Complements `AL_Init()`. Comprehensive cleanup of locked memory.

### AL_Reset
- **Signature:** `void AL_Reset(void)`
- **Purpose:** Resets Adlib card to quiet state via hardware register writes.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Writes to card registers; calls `AL_StereoOn()`, `AL_FlushCard()`
- **Calls:** `AL_SendOutputToPort()`, `AL_StereoOn()`, `AL_FlushCard()`
- **Notes:** Sets AM/VIB/Rhythm register (0xBD) to 0. Calls `AL_FlushCard()` for left/right ports if stereo.

### AL_FlushCard
- **Signature:** `void AL_FlushCard(int port)`
- **Purpose:** Silences all voices on a card port by clearing envelope and maximizing attenuation.
- **Inputs:** Port address (0x388 or 0x38A)
- **Outputs/Return:** None
- **Side effects:** Writes multiple registers via `AL_SendOutputToPort()`
- **Calls:** `AL_SendOutputToPort()`
- **Notes:** Sets fast envelope (0xFF), disables note-on bit, applies maximum attenuation (0xFF) to both operators per voice.

### AL_ReserveVoice / AL_ReleaseVoice
- **Signature:** `int AL_ReserveVoice(int voice)` / `int AL_ReleaseVoice(int voice)`
- **Purpose:** Mark/unmark voices as reserved (unavailable to MIDI driver).
- **Inputs:** Voice number (0–8)
- **Outputs/Return:** `AL_Ok`, `AL_Warning`, or `AL_Error`
- **Side effects:** Updates `VoiceReserved[]`, modifies `Voice_Pool`; `AL_ReserveVoice()` may call `AL_NoteOff()` if voice active
- **Calls:** `DisableInterrupts()`, `AL_NoteOff()`, `LL_Remove()`, `LL_AddToTail()`, `RestoreInterrupts()`
- **Notes:** Interrupt-safe. Allows coexistence with other FM drivers using disjoint voices.

### AL_DetectFM
- **Signature:** `int AL_DetectFM(void)`
- **Purpose:** Detects presence of Adlib-compatible card via timer test.
- **Inputs:** None
- **Outputs/Return:** Boolean (1 if present, 0 if absent)
- **Side effects:** Writes/reads hardware registers; bypassed if `NO_ADLIB_DETECTION` user parameter set
- **Calls:** `AL_SendOutputToPort()`, `inp()`, `USER_CheckParameter()`
- **Notes:** Uses OPL2 timer flag changes to confirm card presence.

### AL_RegisterTimbreBank
- **Signature:** `void AL_RegisterTimbreBank(unsigned char *timbres)`
- **Purpose:** Replaces default timbre bank with user-supplied timbres.
- **Inputs:** Pointer to 256 × 13-byte packed timbre data
- **Outputs/Return:** None
- **Side effects:** Unpacks and updates global `ADLIB_TimbreBank[]` array
- **Calls:** None
- **Notes:** Each timbre: 2 SAVEK + 2 Level + 2 Env1 + 2 Env2 + 2 Wave + 1 Feedback + 1 Transpose + 1 Velocity = 13 bytes. Simple loop unpack.

## Control Flow Notes

**Initialization:** `AL_Init()` → `DPMI_Lock()` (memory regions) → card detection (Sound Blaster BLASTER_GetEnv()) → port assignment → `AL_CalcPitchInfo()` → `AL_Reset()` → `AL_ResetVoices()`.

**Runtime MIDI:** User calls `AL_NoteOn()`, `AL_NoteOff()`, `AL_ControlChange()`, `AL_ProgramChange()`, `AL_SetPitchBend()` → voice allocation/deallocation → timbre/volume/pitch register writes.

**Shutdown:** `AL_Shutdown()` → `AL_StereoOff()` → `AL_ResetVoices()` → `AL_Reset()` → `DPMI_Unlock()` (all regions).

**Memory locking:** Functions from `AL_SendOutputToPort()` to `AL_LockEnd()` are locked for real-time interrupt-safe operation (marked by `AL_LockStart` define).

## External Dependencies

- **Notable includes:** `<conio.h>` (I/O), `<dos.h>` (DOS), `<stdlib.h>` (standard library)
- **Local headers:** `dpmi.h` (memory locking), `interrup.h` (interrupt control), `sndcards.h` (card types), `blaster.h` (Sound Blaster config), `user.h` (user params), `al_midi.h`, `_al_midi.h` (MIDI types), `ll_man.h` (linked list)
- **Defined elsewhere:** `ADLIB_TimbreBank[]`, `ADLIB_PORT` constant, `outp()`/`inp()` system I/O, `LL_Remove()`, `LL_AddToTail()` linked-list ops, `hibyte()` macro, MIDI constants (`MIDI_VOLUME`, `MIDI_PAN`, etc.), voice/channel constants (`NUM_VOICES`, `NUM_CHANNELS`, `MAX_NOTE`, etc.), error codes (`AL_Ok`, `AL_Error`, `AL_VoiceNotFound`), `DPMI_*()` functions, `BLASTER_*()` functions, `DisableInterrupts()`, `RestoreInterrupts()`, `USER_CheckParameter()`.
