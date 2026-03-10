# rott/snd_reg.h — Enhanced Analysis

## Architectural Role

This file serves as the **sound-to-driver binding layer** between gameplay logic and the multi-backend audio subsystem (MUSE/AdLib/Blaster). Every time game code triggers a sound effect, it references an enum value here; the `sounds[]` lookup table then translates that into driver-specific parameters. This design decouples game logic from audio implementation details—critical in the 1990s when supporting multiple audio cards (AdLib, Sound Blaster, Roland) was essential.

## Key Cross-References

### Incoming (who depends on this file)
- **Gameplay systems**: UI menus (cursor movement, selection), player actions (attacks, damage, pickup), actor AI (guards seeing player, death cries), level progression (start/end), environment interactions (doors, explosions, traps)
- The enum `digisounds` is referenced throughout game code wherever a sound effect needs to be played; the pattern suggests `PlaySound(D_MENUFLIP)` or similar calls
- No direct function calls—purely enum-based dependency through the `sounds[]` array

### Outgoing (what this file depends on)
- **MUSE backend enum values** (`MUSE_MENUFLIPSND`, `MUSE_PLAYERDYINGSND`, etc.)—defined in audio driver headers (likely `audiolib/`)
- **Sound priority constants** (`SD_PRIOMENU`, `SD_PRIOGAME`, `SD_PRIOPHURT`, `SD_PRIOREMOTE`, etc.)—audio mixer arbitration levels
- **Playback flags** (`SD_PITCHSHIFTOFF`, `SD_PLAYONCE`, `SD_WRITE`)—driver-specific voice behavior
- **`MAXSOUNDS` constant**—array bound (defined in a sound/audio header)

## Design Patterns & Rationale

**Priority-based mixer queue**
- Sounds are assigned priority tiers: menu > game > secondary > environmental
- Rationale: Limited audio voice channels (4–8 typical on era hardware). Menu UI must not be drowned by weapon fire; boss battle sounds trump ambient wind.
- This is pure 1990s resource constraint design—modern engines manage dynamic voice allocation differently.

**Sound pooling / reuse**
- Multiple digital sounds often map to the same MUSE backend ID (e.g., `D_QUIT1SND` through `D_QUIT7SND` all map to `MUSE_SELECTSND`)
- Rationale: The backend may synthesize these from a single digital waveform, or the game designers deemed them interchangeable for mixing purposes. Reduces ROM/memory footprint.
- This is idiomatic to fixed-backend audio where you curate sound tables carefully.

**Category-driven organization**
- Sounds grouped by gameplay context: menus, weapons, player, actors, environment, secrets
- Rationale: Ease of maintenance; clear ownership (weapon programmer adds weapon sounds here, enemy AI programmer adds monster sounds here)
- Doubles as a form of documentation—you can scan the enum and understand what audio surface area the game exposes.

**Static lookup at compile time**
- `sounds[]` is fully initialized and immutable; no dynamic loading or hot-swapping
- Rationale: DOS/early Windows environments had unpredictable runtime memory. Compile-time data tables are guaranteed to load correctly.

## Data Flow Through This File

```
Game logic          Lookup & Configuration      Audio Driver
─────────────────────────────────────────────────────────────
Call PlaySound()
  │
  └─> enum value D_MENUFLIP
       │
       └─> sounds[D_MENUFLIP] ──> {MUSE_MENUFLIPSND, flags, priority}
                                   │
                                   ├─> MUSE layer selects backend (AdLib/Blaster)
                                   ├─> Priority throttles competing sounds
                                   └─> Flags configure playback (no pitch shift, etc.)
                                       │
                                       └─> Audio hardware plays sample/synth
```

**Key insight**: This file is a **pivot point**. Game code never needs to know whether AdLib or Sound Blaster is running; it just says "D_EXPLOSIONSND" and the lookup table handles the platform-specific binding.

## Learning Notes

**What a developer learns from this file:**
1. **1990s audio architecture**: A snapshot of how constrained-resource game audio worked before streaming and dynamic mixing became standard.
2. **Enum-driven configuration**: Using enums as table indices is fast and safe—compile-time verification that you don't reference invalid IDs.
3. **Priority as a design tool**: Not just a performance knob; it encodes the designer's intent: which sounds matter more in a given context.
4. **Multi-backend support**: The MUSE abstraction layer (present across the `audiolib/` hierarchy) shows how games isolated themselves from hardware variation.

**How modern engines differ:**
- **Dynamic playback**: Modern engines (Wwise, FMOD, Unreal Audio) manage voice allocation at runtime, with priority re-evaluation every frame.
- **Streaming**: This file assumes all sounds are sampled and in ROM; modern engines stream audio to save memory.
- **Soft synth backends**: Modern audio engines use software synthesizers; AdLib/Roland MIDI synthesis is obsolete.
- **Audio events**: Modern engines use hierarchical event systems, not flat enums; a "gunshot" event might vary by weapon, distance, surface, etc.

**Idiomatic patterns:**
- Priority-based voice arbitration was *the* standard in 1990s game audio.
- Static initialization tables (as opposed to dynamic loading) were safer on DOS/Windows 3.1 memory models.
- Category-driven organization predates modern tagging systems.

## Potential Issues

1. **Inconsistent mappings**: Some entries reuse MUSE IDs (e.g., all QUIT sounds → `MUSE_SELECTSND`); others have unique mappings. No comment explains when reuse is intentional vs. placeholder.

2. **Commented-out sounds**: Many entries (e.g., `D_WALK1SND`, `D_WALK2SND`, `D_SNEAKYSPRINGFSND`) are commented out. No cleanup or removal suggests technical debt or uncertainty about which sounds are live.

3. **Placeholder MUSE mappings**: Some entries map to `MUSE_LASTSOUND` (e.g., remote actor sounds), which feels like a sentinel/error value. Suggests incomplete feature implementation.

4. **No inline documentation of intent**: Flags like `SD_PITCHSHIFTOFF` lack comments explaining *why* that sound shouldn't be pitch-shifted (e.g., "menu sounds are UI, not effects").

---

**Summary**: `snd_reg.h` is a straightforward but revealing artifact of 1990s game audio design—a static lookup table that bridges game logic and hardware-specific audio backends. Its organization by category and priority-based arbitration reflect both the technical constraints of the era and smart architectural choices that decouple gameplay from platform details.
