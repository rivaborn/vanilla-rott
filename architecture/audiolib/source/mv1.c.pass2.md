# audiolib/source/mv1.c — Enhanced Analysis

## Architectural Role

**Multivoc** is the **core real-time audio mixing engine** of the audio subsystem, sitting between high-level sound playback requests and hardware-specific sound card drivers (Sound Blaster, Pro Audio Spectrum, Sound Source). It implements software-based multichannel voice mixing for DOS, handling the computation-intensive task of blending multiple VOC-format sounds at fixed-point precision into pre-allocated DMA buffers. The interrupt-driven architecture allows the game engine to fire-and-forget sound playback without blocking on audio operations—critical for real-time game responsiveness on DOS hardware.

## Key Cross-References

### Incoming (who depends on this file)
- **Game code** (implied): Calls `MV_Play()` and `MV_Play3D()` for sound effects; calls `MV_Kill()`, `MV_SetPan()`, `MV_SetVolume()` for runtime control
- **Audio subsystems**: ADLIBFX, AL_MIDI, and other sound modules would use Multivoc or its abstractions for voice management
- **Initialization chain**: `MV_Init()` is called once during engine startup (likely from audio init logic)

### Outgoing (what this file depends on)
- **Hardware drivers** (audiolib/source/blaster.c, pas16.c, sndsrc.c): Calls `BLASTER_Init()`, `BLASTER_BeginBufferedPlayback()`, `BLASTER_SetMixMode()`, `BLASTER_GetPlaybackRate()`, etc.—driver-specific implementations for Sound Blaster, Pro Audio Spectrum, Sound Source
- **DPMI layer** (dpmi.h): Manages DOS memory allocation and memory locking for real-time safety
- **Pitch module** (pitch.h): `PITCH_GetScale()` and `PITCH_LockMemory()` for playback rate calculation
- **Linked list utilities** (ll_man.h): Voice list management via LL_AddToTail, LL_Remove macros
- **User memory hooks** (usrhooks.h): Custom allocation if standard malloc unsuitable

## Design Patterns & Rationale

**1. Double Buffering (pages MV_PlayPage / MV_MixPage)**
- Prevents audible glitches: one buffer plays while the next is being mixed
- Invoked each ISR in `MV_ServiceVoc()` to toggle pages atomically
- Rationale: CPU cannot reliably mix in real-time fast enough for continuous playback at ISR boundaries without tearing

**2. Interrupt-Driven Architecture**
- Sound card fires ISR on each buffer boundary (4–64ms typical)
- `MV_ServiceVoc()` delegates mixing to `MV_PrepareBuffer()`, which must complete before next interrupt
- Memory locking (`MV_LockMemory()` region: MV_Mix8bitMono → MV_LockEnd) prevents page faults during ISR
- Rationale: DOS lacks preemptive scheduling; manual memory locking avoids VM page faults that would exceed ISR deadline

**3. Voice Pool + Priority Allocation**
- `VoicePool` (free voices) vs. `VoiceList` (playing voices) using linked-list containers
- `MV_AllocVoice()` implements **voice stealing**: if pool empty, steals the lowest-priority active voice
- Rationale: Limits memory overhead; ensures high-priority sounds (e.g., enemy fire) always play even if voice budget exceeded

**4. Lookup Table-Based Mixing**
- **Volume tables** (`MV_8BitVolumeTable`, `MV_16BitVolumeTable`): Pre-computed (volume, sample_byte) → output_byte mappings
- **Panning tables** (`MV_PanTable`): Pre-computed left/right volume pairs for 3D positions
- **Harsh clipping tables** (`HarshClipTable`, `HarshClipTable16`): Soft clipping curve via LUT instead of conditional logic
- Rationale: Lookup is O(1), avoids expensive multiplies/conditional branches in tight ISR loop; enables smooth audio without saturation artifacts

**5. Fixed-Point Arithmetic (16.16 format for playback rate)**
- Position: `voice->position` is 16.16 fixed-point (integer index in upper 16 bits, sub-sample precision in lower)
- Rate scaling: `voice->RateScale` is incremented per sample to handle arbitrary sampling rates and pitch shifting
- Rationale: Integer-only; no FPU (rare in DOS era); microsecond-level precision for pitch without floating-point overhead

**6. VOC Format Parsing (stateful block streaming)**
- `MV_GetNextVOCBlock()` is called when playback position exhausts current block
- Handles loop markers (block 6/7), silence blocks, format changes, new format (block 9)
- Supports only 8-bit mono PCM; skips packed/stereo variants
- Rationale: Legacy compatibility; many DOS games ship sounds in VOC; streaming avoids loading entire file into memory

## Data Flow Through This File

```
Game Engine
    │
    ├─→ MV_Init(soundcard, rate, voices, mode)
    │   ├─ Allocate DOS memory (DMA-safe)
    │   ├─ Create voice pool
    │   ├─ Lock mixing code/state into physical RAM
    │   ├─ Call BLASTER_Init() / PAS_Init() / SS_Init()
    │   └─ MV_StartPlayback() → register MV_ServiceVoc() as ISR callback
    │
    ├─→ MV_Play(sound_ptr, pitch, vol, pan_left, pan_right, priority, callback)
    │   ├─ MV_AllocVoice(priority) → may steal lower priority voice
    │   ├─ MV_GetNextVOCBlock() → parse first VOC block
    │   ├─ Add to VoiceList (with interrupts disabled)
    │   └─ Return handle
    │
    └─→ [Game runtime]
        │
        ├─ MV_SetPan(handle, vol, left, right)
        ├─ MV_SetPitch(handle, pitch_offset) → updates RateScale
        ├─ MV_Kill(handle) → stop voice, invoke callback
        │
        └─ Sound Card Interrupt (every ~20ms)
           │
           ├─→ MV_ServiceVoc() [Real-time ISR]
           │   ├─ MV_DeleteDeadVoices() → move finished voices to pool, invoke callbacks
           │   ├─ Swap MV_PlayPage ↔ MV_MixPage
           │   └─ MV_PrepareBuffer(next_page)
           │       ├─ ClearBuffer_DW() with silence fill (0x00 or 0x8000)
           │       ├─ For each voice in VoiceList:
           │       │   ├─ Select mix function based on format (8/16-bit, mono/stereo)
           │       │   └─ MV_Mix8bitMono() / Stereo / 16bitUnsignedMono() / Stereo()
           │       │       ├─ Read sample at fixed-point position
           │       │       ├─ Lookup volume via MV_8BitVolumeTable or MV_16BitVolumeTable
           │       │       ├─ Accumulate into output sample
           │       │       ├─ Soft-clip via HarshClipTable[...] lookup
           │       │       ├─ If position exhausted: MV_GetNextVOCBlock()
           │       │       └─ Advance position by RateScale
           │       └─ [Buffer ready for DMA]
           │
           └─ DMA transfers MV_PlayPage to sound card DAC
```

**Key state mutations during ISR**: `voice->position`, `voice->Playing`, `voice->sound`, `voice->length`, `VoiceList` (via MV_DeleteDeadVoices)

## Learning Notes

**Idiomatic to DOS era / this engine:**
- **Real-time constraints are explicit**: Memory locking, interrupt-safe code, pre-allocated buffers; modern engines abstract this away behind OS/frameworks
- **Fixed-point arithmetic**: No FPU; all pitch/rate calculations use 16.16 shifts instead of floats
- **VOC format** (1990s): Obsolete; modern engines use MP3, OGG, OPUS; VOC is simple but limited (8-bit mono only in this implementation)
- **Lookup tables for DSP**: Clipping and panning via LUT; modern GPUs vectorize this; modern CPUs have fast multiplies so branches become preferable
- **Stateful block streaming** in `MV_GetNextVOCBlock()`: Assumes single playback thread; no re-entrancy

**Cross-engine comparison:**
- **ECS pattern**: This is implicit—voices are entities, mixing is a system; no explicit ECS framework
- **Voice stealing**: Common in legacy hardware (MIDI synths, Game Boy); modern engines have higher voice budgets or dynamic allocation
- **Panning via table lookup**: Modern engines compute pan gains in real-time; lookup trades memory for CPU (valuable in 1994)
- **Harsh clipping LUT**: Soft clipping; modern audio often uses multiband compression or better dithering

## Potential Issues

1. **Race condition on `VoiceList` modification**: `VoiceList` is marked `volatile` but accessed without atomic ops in `MV_Play()` and `MV_Kill()`. Critical sections use `DisableInterrupts() / RestoreInterrupts()`, but if ISR re-enters during node removal, traversal could be undefined. Modern fix: RCU or lock-free list.

2. **Harsh clipping table indexing assumptions**: Lines ~270, ~380 use indices like `HarshClipTable[4*256 + samp - 0x80]`. The offset `4*256` assumes a specific layout; if table allocation changes, silent corruption occurs. No bounds checking.

3. **Fixed-point overflow in position**: `position` is 32-bit (16.16 FP); for a long sound at high rate, could wrap. No wraparound handling visible (though unlikely in practice for VOC files < 2GB).

4. **VOC block 9 (new format) support incomplete**: Code only accepts BitsPerSample=8, Channels=1, Format=0. Stereo or 16-bit VOC blocks silently fail (voice stops playing). Unclear if intentional or oversight.

5. **No error recovery in `MV_GetNextVOCBlock()`**: Malformed VOC data could cause buffer overrun or infinite loop (e.g., block size 0 or cyclic loop markers). Assumes input is well-formed.

---

**Sources used**: First-pass analysis, file content (lines 1–2132), cross-reference function map excerpt, general DOS audio architecture knowledge.
