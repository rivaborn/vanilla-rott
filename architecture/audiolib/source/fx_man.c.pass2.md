Looking at the cross-reference context and first-pass analysis, I'll now write the enhanced architectural analysis for `fx_man.c`.

# audiolib/source/fx_man.c — Enhanced Analysis

## Architectural Role

FX_MAN acts as the **device-independent audio facade** for the game engine (rt_*.c, cin_*.c modules visible in cross-reference), abstracting four distinct audio stacks: digitized PCM (Sound Blaster, PAS16, SoundScape, Ultrasound, SoundSource) via the MULTIVOC mixer, MIDI/FM synthesis (not detailed in first pass but implied by GenMidi/SoundCanvas references), and legacy AdLib support (see cross-reference: ADLIBFX_*). The file is the sole entry point for the game to request audio playback without knowing which sound card is installed or how to initialize it.

## Key Cross-References

### Incoming (who depends on this file)
- **Game engine modules** (rt_*.c, cin_*.c): Call FX_Init() at startup, FX_Play* during gameplay, FX_StopSound/FX_StopAllSounds for cleanup, FX_SetVolume for menu/settings
- **Cinematic system** (cin_*.c): Likely calls FX_PlayVOC/FX_PlayWAV for cutscene audio, FX_StartRecording for voice capture
- **Menu system** (rt_menu.h references CP_SoundSetup): Uses FX_SetupCard, FX_GetBlasterSettings to configure audio in control panel
- **No direct cross-reference entries** in excerpt, but implied through initialization and state globals FX_SoundDevice, FX_Installed

### Outgoing (what this file depends on)
- **Device drivers:** BLASTER_* (blaster.c/h), PAS_* (pas16.h), SOUNDSCAPE_* (sndscape.h), GUSWAVE_* (guswave.h), SS_* (sndsrc.h) — called directly via switch on FX_SoundDevice
- **Core mixer:** MV_* functions (multivoc.h) — ALL playback, stopping, voice control delegated here; FX_* just wraps MV_* in device routing
- **Memory management:** LL_LockMemory / LL_UnlockMemory (ll_man.h) — brackets audio system lifetime to prevent DOS paging
- **User input:** USER_CheckParameter (user.h) — checks ASSVER environment variable for version display

## Design Patterns & Rationale

**Facade + Strategy Pattern:** FX_MAN presents a unified API while FX_SoundDevice acts as a "strategy selector." At init time, one strategy (driver) is locked in; thereafter, all calls route through it. This avoids runtime polymorphism overhead in 1995-era C.

**Error Propagation Indirection:** FX_ErrorString (called by game) does NOT directly query BLASTER_Error or MV_Error; instead, it delegates *via FX_ErrorCode global*. This decouples error retrieval from error generation and enables delayed error reading (user might check error after calling FX_PlayVOC).

**Why MULTIVOC delegation?** Rather than replicating voice mixing in fx_man, the file trusts MV_* to be the canonical mixer for all digitized audio. Device-specific code only touches hardware init and volume control (where some cards have hardware mixers, others don't).

## Data Flow Through This File

1. **Initialization:** `FX_Init(card, params)` → `LL_LockMemory()` → `MV_Init(card, ...)` → `FX_Installed = TRUE`. DPMI memory lock is critical to prevent DOS IRQ handlers from being paged out.
2. **Playback Request:** `FX_PlayVOC(buffer, params)` → checks FX_SoundDevice → calls `MV_PlayVOC(...)` → returns voice handle. FX_MAN adds no logic; it's purely a device dispatcher.
3. **Voice Control:** `FX_SetPan/Pitch/Frequency(handle, ...)` → `MV_SetPan/Pitch/Frequency(...)`. No device routing needed; MULTIVOC abstracts the hardware.
4. **Volume (device-specific):** `FX_SetVolume(vol)` → tries Sound Blaster hardware mixer (if available), falls back to `MV_SetVolume`. PAS16 and Gus follow similar fallback chains.
5. **Shutdown:** `FX_Shutdown()` → `MV_Shutdown()` → `LL_UnlockMemory()` → `FX_Installed = FALSE`. Must mirror FX_Init's memory locking.

## Learning Notes

**DOS Audio Legacy:** This code reflects 1995 sound card fragmentation: Sound Blaster dominated but ProAudio Spectrum, SoundScape, Ultrasound, and SoundSource each had different interrupt/DMA configurations. Modern engines use a single API (ALSA, PulseAudio, Core Audio, Wasapi) and let the OS route. Here, the *game* must know which card to initialize.

**DPMI Memory Locking:** LL_LockMemory locks the audio mixer in physical RAM so DOS IRQ handlers (audio IRQs fire ~44KHz) never page fault. This is invisible on modern systems but essential in protected mode DOS.

**Hardware Mixer Abstraction Asymmetry:** Sound Blaster and PAS16 *may* have hardware volume control; if it fails, FX_MAN falls back to software (MV_SetVolume). Ultrasound uses GUSWAVE driver exclusively. This inconsistency is intrinsic to era hardware, not a design flaw.

**Callback Pattern for Voice Completion:** Games register a callback via FX_SetCallBack; MULTIVOC invokes it when a voice finishes. This allows the game to queue the next sound without polling. FX_MAN merely forwards the callback to MV_SetCallBack.

## Potential Issues

- **No recovery from partial init failure:** If `LL_LockMemory()` succeeds but `MV_Init()` fails, `LL_UnlockMemory()` is called, leaving FX_Installed = FALSE. However, if a prior FX_Init call set FX_Installed = TRUE and a second FX_Init fails mid-way, the first call's lock is released prematurely. No observed issue in provided code because FX_Init calls FX_Shutdown first, but the pattern is fragile.
- **Device type assumed stable:** FX_SoundDevice is set once and never validated during playback. If a driver crash (rare but possible with buggy hardware) resets the device, FX_* calls may dereference stale pointers. No defensive checks in playback paths.
- **MIDI device support incomplete:** GenMidi/SoundCanvas/WaveBlaster initialize with zero capabilities and no delegate calls. Games attempting FX_PlayVOC on a MIDI-only card will silently fail in MV_Init, confusing users. Could benefit from early error reporting in FX_SetupCard.
