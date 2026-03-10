# audiolib/source/guswave.c — Enhanced Analysis

## Architectural Role

GUSWAVE is a hardware device driver and voice manager for the Gravis Ultrasound audio card, positioned as one peer in audiolib's family of audio hardware adapters (ADLIBFX, BLASTER, AL_MIDI/AWE32 for MIDI). It provides the polyphonic digitized sound playback substrate, translating high-level `PlayVOC()`/`PlayWAV()` requests into GUS hardware voice allocation and ISR-driven streaming. Unlike the MIDI subsystems (which work synth-side), GUSWAVE manages raw PCM data in hardware-limited onboard memory, making it both a bridge layer and resource manager.

## Key Cross-References

### Incoming (who depends on this)
- **Game engine layer** (rott/rt_*): Likely calls GUSWAVE functions indirectly via a higher-level mixer/audio abstraction (not fully visible in excerpt, but standard architecture)
- **MIDI subsystem** (gusmidi.h): Checks `GUSMIDI_Installed` to coordinate voice usage; GUS memory is shared between digital and MIDI playback
- **Error routing** (multivoc.h): Error codes map to `multivoc` error domain, suggesting GUSWAVE is wrapped in or coordinated with a multivocal audio mixer

### Outgoing (what this depends on)
- **GUS hardware layer** (newgf1.h): `gf1_play_digital()`, `gf1_stop_digital()`, `gf1_dig_set_freq/pan/vol()`, `gf1_malloc()`
- **Pitch subsystem** (pitch.h): `PITCH_GetScale()` maps semitone offsets to hardware sample-rate multipliers
- **Linked list utilities** (ll_man.h): Generic linked list operations for VoicePool/VoiceList management
- **Interrupt control** (interrup.h): `DisableInterrupts()`, `RestoreInterrupts()` protect shared state during ISR interactions
- **Debug output** (debugio.h): Optional verbose logging in ISR context via `DB_printf()`, `DB_PrintNum()`
- **User parameter checking** (user.h): Command-line parameter validation during init

## Design Patterns & Rationale

**1. Interrupt-Driven Callback Streaming**  
GUSWAVE_CallBack() runs in ISR context, pulling audio blocks on-demand via polymorphic `GetSound()` methods. This avoids polling and reduces CPU overhead—essential when GUS onboard memory (typically 1–4 MB in 1994) necessitates chunked streaming rather than full-file buffering.

**2. Pre-Allocated Voice Pool**  
VoicePool/VoiceList are static linked lists; allocation never happens inside ISR. Voice deallocation is deferred to ISR completion callback to maintain interrupt safety—a pattern uncommon in modern engines (which use lock-free queues or thread-safe allocators).

**3. Priority-Based Voice Preemption**  
When VoicePool is exhausted, `GUSWAVE_AllocVoice()` scans VoiceList for the lowest-priority active voice and silently preempts it. This allows game-critical sounds (dialogue, UI) to preempt background music without explicit user code. Modern middleware (FMOD, Wwise) handle this via event priority queuing; embedded in the driver here due to era constraints.

**4. Format-Specific Streaming via Function Pointers**  
Each voice's GetSound() points to one of three block-fetchers (VOC, WAV, or demand-feed). This is lightweight polymorphism avoiding class overhead—idiomatic for 1994 C.

**5. Symmetric Pan Lookup Table**  
GUSWAVE_PanTable[32] mirrors around center (index 15/16), enabling O(1) pan angle → hardware value mapping. The swap-stereo flag allows left/right reversal without recomputing.

## Data Flow Through This File

```
Entry:  PlayVOC() / PlayWAV() / StartDemandFeedPlayback()
    ↓
AllocVoice()  [may preempt lower-priority voice]
    ↓
Initialize voice node (sampling rate, bits, GetSound function pointer)
    ↓
GUSWAVE_Play()  →  gf1_play_digital()  [register ISR callback]
    ↓
Voice moves:  VoicePool → VoiceList
    ↓
[ISR context, repeated until sound exhausted]
    ↓
GUSWAVE_CallBack(DIG_MORE_DATA)  →  voice->GetSound()  →  fetch next block
    ↓
Return buffer pointer & size to hardware; hardware plays sample
    ↓
[When data exhausted or silence returned]
    ↓
GUSWAVE_CallBack(DIG_DONE)  
    →  Mark Active=FALSE, Playing=FALSE
    →  Remove from VoiceList, add to VoicePool
    →  Invoke user callback
    ↓
Exit:  Voice returns to pool, handle becomes invalid
```

**Control points during playback:**  
- `SetPitch()`, `SetPan3D()`, `SetVolume()` update hardware parameters in-flight via `gf1_dig_set_freq/pan/vol()`
- `Kill()` stops immediately (unlike natural completion); deallocation still deferred to ISR
- Master volume scales all per-voice volumes multiplicatively

## Learning Notes

**Era-Specific Idioms (1994–1995):**
1. **Onboard memory as bottleneck**: GUS had 256 KB–16 MB of expensive DRAM. Streaming in ~256–1024 byte chunks forced ISR-driven fetching; modern cards have multi-GB and use DMA or thread-driven buffering.
2. **ISR as audio engine**: Callback is the audio loop; no dedicated audio thread. Modern engines use dedicated threads (or GPU compute) to decouple from CPU interrupt latency.
3. **Format parsing in driver**: VOC/WAV parsing embedded; no separate codec abstraction. Modern audio stacks delegate to codec libraries (libvorbis, libflac, libopus).
4. **Voice as hardware entity**: Each voice = one GUS hardware voice (GF1 voice slot); modern engines mix all voices to a stereo bus via software.
5. **Pan table as precomputed optimization**: 32-entry table avoids floating-point math in ISR; modern engines use inline HRTF or 3D positional audio math.

**Curious Design Choices:**
- **VOC vs. WAV parity mismatch**: VOC parser implements repeat blocks (6–7) with loop state machine; WAV parser ignores loop points ("loops not currently implemented"). Asymmetry suggests VOC was the primary format.
- **Silent buffer constants**: 0x80 (unsigned PCM midpoint) for 8-bit, 0x0000 for 16-bit signed. Buffers are 1024 bytes (256 samples at typical rates), suggesting fixed block size for GUS DMA alignment.
- **Voice handle monotonic counter**: Increments forever without wraparound; on multi-day uptime with millions of allocations, could theoretically overflow (though comparison with int stops before INT_MAX).
- **Stereo channels silently skipped**: VOC stereo blocks jump over without error; WAV stereo is attempted but loop pointers left NULL. Modern drivers would error or downmix explicitly.

**Connections to Modern Concepts:**
- GUSWAVE_AllocVoice + preemption → Audio engine voice prioritization (FMOD Studio, Wwise)
- Demand-feed callbacks → Streaming codec iterators (AAC-LC chunk fetchers, Vorbis packet demux)
- Pan table → HRTF lookup or 3D audio positioning
- ISR callback loop → Real-time audio thread (pthread, OS::Thread)

## Potential Issues

1. **Voice Handle Overflow**: `GUSWAVE_VoiceHandle` monotonically increments without wraparound. Malicious or extremely long-running sessions (millions of allocations) could overflow, causing handle collision or undefined behavior in handle lookups.

2. **VOC/WAV Loop Asymmetry**: VOC repeat blocks are parsed and looped; WAV loop points are explicitly zeroed in `GUSWAVE_PlayWAV()`. Porting VOC audio to WAV may silently break intended looping behavior without compiler warning.

3. **Silent Codec Format Skipping**: When VOC encounters stereo or packed blocks, the parser skips them silently without error or log. Malformed or unusual VOC files could play truncated/corrupted audio without diagnostic output.

4. **ISR Race on Voice Deallocation**: `GUSWAVE_Kill()` calls `gf1_stop_digital()` but returns immediately; deallocation (VoiceNode return to pool) happens asynchronously in ISR callback. If client code reuses the voice handle before ISR fires, handle comparison may succeed on a stale/reallocated node (though Active flag should catch this).

5. **Debug Callback Overhead in ISR**: `GUSWAVE_DebugCallBack()` logs via `DB_printf()` in interrupt context. If accidentally left enabled in production, could cause real-time audio glitches due to ISR latency.

6. **Silence Buffer Size Inconsistency**: Callback returns 256 bytes of silence, but GUS_Silence8[1024] and GUS_Silence16[512] suggest 1024-byte blocks. If hardware expects full buffer size, repeated 256-byte silence blocks could cause underrun gaps.

7. **No Sample-Rate Conversion**: Pitch changes via playback rate scaling (`gf1_dig_set_freq`), not resampling. Incompatible hardware sample rates (e.g., VOC at 22 kHz on GUS expecting 44 kHz) could play at wrong pitch without automatic conversion.
