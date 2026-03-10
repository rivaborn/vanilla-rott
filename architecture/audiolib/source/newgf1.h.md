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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| load_os | struct | Card initialization config: voice count, forced I/O base, IRQ, DMA channel |
| patchheader | struct | Patch file header: Gravis ID, description, instrument/voice/channel counts, wave forms, master volume |
| instrumentdata | struct | Per-instrument metadata: name, size, layer count |
| layerdata | struct | Per-layer metadata: duplicate flag, size, sample count |
| patchdata | struct | Per-waveform on-disk data: loop points, frequencies, sample rate, envelope/tremolo/vibrato parameters, modes |
| wave_struct | struct | Runtime waveform with playback state: loop bounds, frequencies, accumulator/position registers, envelope/modulation params |
| patchinfo | struct | Combined patch header and instrument data |
| patch | struct | Complete loaded instrument: layer count, wave arrays per layer, detune value |
| gf1_dma_buff | struct | DMA buffer descriptor: virtual pointer and physical address |
| gf1_sound | struct | Sound instance for digital playback: memory position, loop points, type flags |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| gf1_log_table | long[12] | extern | Logarithmic table for vibrato and pitch bend frequency calculations |
| gf1_linear_volumes | char | extern | Flag controlling linear vs. logarithmic volume envelope mode |
| gf1_dig_use_extra_voice | char | extern | Flag to allocate extra voice for digital audio playback |

## Key Functions / Methods

### gf1_allocate_voice
- Signature: `int gf1_allocate_voice(int priority, void (RFAR *steal_notify)(int))`
- Purpose: Allocate a hardware voice for audio playback with priority-based preemption
- Inputs: priority (voice priority level), steal_notify (callback when voice is stolen by higher priority)
- Outputs/Return: Voice handle (integer ≥ 0) or error code (< 0, e.g., NO_MORE_VOICES)
- Side effects: Modifies global voice allocation table; may invoke steal_notify if preemption occurs
- Calls: Not inferable from this file
- Notes: Core voice management; hardware typically supports 14–32 voices; voice stealing is priority-based

### gf1_play_digital
- Signature: `int gf1_play_digital(unsigned short priority, unsigned char RFAR *buffer, unsigned long size, unsigned long gf1_addr, unsigned short volume, unsigned short pan, unsigned short frequency, unsigned char type, struct gf1_dma_buff RFAR *dptr, int (RFAR *callback)(int, int, unsigned char RFAR * RFAR *, unsigned long RFAR *))`
- Purpose: Start DMA playback of digital audio buffer on GF1 hardware
- Inputs: priority, buffer pointer, size (bytes), GF1 DRAM address, volume, pan, frequency, type flags (8/16-bit, stereo, etc.), DMA buffer descriptor, streaming callback
- Outputs/Return: Voice handle or error code
- Side effects: Allocates voice, initiates DMA transfer, configures GF1 playback registers
- Calls: gf1_allocate_voice (inferred)
- Notes: Callback invoked on DMA completion for streaming; supports looping and bidirectional loop modes; type flags control data format and preload behavior

### gf1_midi_note_on
- Signature: `void gf1_midi_note_on(struct patch RFAR *patch, int priority, int note, int velocity, int channel)`
- Purpose: Trigger MIDI note playback using a loaded patch/instrument
- Inputs: patch (loaded instrument), priority (voice priority), note (0–127), velocity (MIDI velocity), channel (MIDI channel)
- Outputs/Return: void
- Side effects: Allocates voice, starts sample playback, applies velocity to volume
- Calls: gf1_allocate_voice, gf1_midi_status_note (inferred)
- Notes: Core MIDI synthesis function; velocity affects initial volume; channel routing enables per-channel control

### gf1_load_patch
- Signature: `int gf1_load_patch(char RFAR *patch_file, struct patchinfo RFAR *patchinfo, struct patch RFAR *patch, struct gf1_dma_buff RFAR *dptr, unsigned short size, unsigned char RFAR *wavemem, int flags)`
- Purpose: Load patch file from disk into GF1 DRAM and populate runtime patch structure
- Inputs: patch_file (file path), patchinfo (header/instrument metadata), patch (runtime structure), dptr (DMA buffer), size (buffer size), wavemem (wave data destination), flags (e.g., PATCH_LOAD_8_BIT)
- Outputs/Return: Success (0) or error code (e.g., FILE_NOT_FOUND, NO_MEMORY)
- Side effects: Opens file, reads waveform data, performs DMA transfer to GF1 DRAM
- Calls: gf1_get_patch_info, gf1_dram_xfer, gf1_open, gf1_read (inferred)
- Notes: Supports multiple layers and waveforms; dptr provides DMA buffer info; wavemem points to GF1 DRAM allocation

### gf1_dram_xfer
- Signature: `int gf1_dram_xfer(struct gf1_dma_buff RFAR *dptr, unsigned long size, unsigned long dram_address, unsigned char dma_control, unsigned short flags)`
- Purpose: Perform DMA transfer of data to/from GF1 DRAM
- Inputs: dptr (DMA buffer), size (bytes), dram_address (GF1 memory address), dma_control (DMA flags: direction, width, IRQ), flags (GF1_RECORD or GF1_DMA)
- Outputs/Return: Success (0) or error code (e.g., DMA_BUSY, DMA_HUNG)
- Side effects: Triggers hardware DMA controller; may block or use interrupt
- Calls: gf1_wait_dma (inferred)
- Notes: Low-level hardware interface; dma_control encodes read/write, data width (8/16-bit), and IRQ enable

## Control Flow Notes
Typical application flow: **(1) Initialize** → gf1_init_ports / gf1_load_os / gf1_detect_card. **(2) Load patches** → gf1_get_patch_info / gf1_load_patch. **(3) Playback** → gf1_allocate_voice / gf1_midi_note_on or gf1_play_digital. **(4) Control** → gf1_midi_change_volume / gf1_sound_frequency / gf1_channel_pitch_bend. **(5) Cleanup** → gf1_unload_patch / gf1_unload_os. DMA callbacks and voice-stealing callbacks drive interrupt-level event handling.

## External Dependencies
- Platform-specific compiler macros (BORLANDC, _MSC_VER) for far pointer qualification (RFAR)
- GF1 hardware memory model (DMA, DRAM, port I/O operations)
- File I/O functions (gf1_open, gf1_read, gf1_close_file) for patch loading
- Callback-based architecture (function pointers) for DMA completion, voice stealing, MIDI events, timers
- MIDI protocol constants and control change definitions
