# rott/snd_shar.h — Enhanced Analysis

## Architectural Role

This file is a **centralized sound asset registry** serving as the bridge between game logic and the audio subsystem. It maps semantic sound events (enums like `D_ATKPISTOLSND`, `D_PLAYERDYINGSND`) to concrete audio resources (digital samples and MUSE synthesizer events), with a priority tier system enforcing allocation of scarce audio voices. This reflects the early 1990s constraint of limited simultaneous playback channels on Sound Blaster and AdLib cards.

## Key Cross-References

### Incoming (who depends on this file)
- **Game logic modules** (`rt_playr.c`, `rt_actor.c`, `rt_game.c`, `rt_menu.h`) issue sound requests using `digisounds` enum IDs
- **Menu system** consumes menu-tier sounds (D_MENUFLIP, D_SELECTSND, D_WARNINGBOXSND) with `SD_PRIOMENU`
- **Weapon/actor systems** trigger gameplay-priority sounds (D_ATKPISTOLSND, D_LOWGUARD1SEESND)
- **Audio playback layer** (consuming ADLIBFX, AL_*, BLASTER_* from audiolib) receives sound records with priority/flags

### Outgoing (what this file depends on)
- **audiolib subsystem**: references to `MUSE_*` constants suggest integration with MUSE synthesizer (likely `audiolib/source/al_midi.c` family)
- **Digital sample asset IDs** (D_* constants) point to a separate sound bank/resource system
- **Sound infrastructure** (`sound_t` typedef, `SD_*` flags/priorities) likely defined in an adjacent sound header (possibly `snd_shar.c` or dedicated header)

## Design Patterns & Rationale

**Priority-Based Resource Arbitration:** The multi-tiered priority system (`SD_PRIOMENU` → `SD_PRIOPSNDS` → `SD_PRIOGAME` → `SD_PRIOPMISS` → `SD_PRIOEXPL` → `SD_PRIOBOSS`) enforces voice allocation under hardware constraints. Boss sounds preempt secrets; weapon fire preempts ambient effects. This is **resource pooling**—the audio driver processes requests in priority order when the voice buffer fills.

**Dual Fallback Architecture:** Each sound maps to both a digital sample (`D_ATKPISTOLSND`) and a MUSE synthesizer event (`MUSE_ATKPISTOLSND`). This allowed runtime selection based on installed hardware—Sound Blaster for samples, AdLib for synthesis. The high count of `D_LASTSOUND` entries suggests Sound Blaster samples were incomplete; MUSE was the reliable fallback.

**Declarative Sound Configuration:** Rather than hardcoding sound playback in gameplay functions, this table decouples game events from audio implementation, allowing iteration without recompiling game logic.

## Data Flow Through This File

1. **Engine Startup:** `sounds[]` table is linked at compile time
2. **Gameplay Event:** Game code calls sound function (e.g., `SD_Play(D_ATKPISTOLSND)`)
3. **Lookup & Arbitration:** Sound system indexes `sounds[D_ATKPISTOLSND]`, extracts priority (`SD_PRIOPGUNS`), flags (`SD_WRITE`)
4. **Voice Allocation:** Audio driver (BLASTER_* or ADLIBFX_* or AL_*) checks available voices; if full, preempts lower-priority sound
5. **Playback:** Selected sample (digital) or synthesizer event (MUSE) plays until completion or manually stopped

## Learning Notes

**Idiomatic to 1994 PC Audio Landscape:**
- Reflects hardware fragmentation (Sound Blaster vs. AdLib) and limited voice polyphony—constraints eliminated by modern OS audio mixing
- Placeholder entries (300+ `D_LASTSOUND` fallbacks) suggest compressed development cycle; unimplemented sounds defaulted to silence
- No per-sound velocity/pitch control visible; pitch-shift flags (SD_PITCHSHIFTOFF) suggest a separate subsystem for pitch variation

**Contrast with Modern Engines:**
- Modern systems (FMOD, Wwise) handle priority dynamically at runtime; audio graphs replace static lookup tables
- This approach embeds hardware knowledge (voice count limits) into data; modern engines abstract it away

**Architectural Insight:**
This file exemplifies **data-driven design** in resource-constrained systems—moving complexity from code to declarative tables allows non-programmers (audio designers) to tune behavior.

## Potential Issues

1. **Placeholder Audio Gaps:** ~50+ entries use `D_LASTSOUND` / `MUSE_LASTSOUND`, indicating incomplete audio asset coverage. Triggering these produces silence or undefined behavior.
2. **No Bounds Validation Visible:** Without seeing the playback code, unclear if invalid `digisounds` enum values are range-checked before indexing `sounds[]`.
3. **Asymmetric Audio:** Systems with only MUSE synthesizer (AdLib-only hardware) may sound different from Sound Blaster systems due to fallback choices; no dynamic quality fallback visible.
