# rott/rt_debug.c — Enhanced Analysis

## Architectural Role

rt_debug.c is the cheat code detection and execution layer that sits between the input subsystem and game state. It acts as a control flow valve: raw keyboard input (from rt_in.h's `LetterQueue`) is pattern-matched against a fixed table of cheat sequences, then dispatched to handler functions that modify game state across multiple subsystems (player, entities, rendering, sound, network, UI). The file serves dual purposes: release-build cheat codes (controlled by `DebugOk` flag) and development-only single-key shortcuts (via `DebugKeys()` when `DEVELOPMENT==1`).

## Key Cross-References

### Incoming (who depends on this file)

- **Main game loop** calls `CheckDebug()` once per frame (likely from rt_game.c or rt_main.c) to poll for active cheat codes
- **Development builds only**: `DebugKeys()` is called from the main loop for frame-by-frame debug control

### Outgoing (what this file depends on)

**Core subsystems modified by cheat handlers:**
- **Player state** (rt_playr.h): `player->z`, `player->health`, `player->keys`, `player->armor`, weapon state
- **Entity/Actor system** (rt_stat.c): `SpawnStatic()`, `MakeStatActive()`, `LASTSTAT` manipulation
- **Game state** (rt_game.h): `gamestate.mapon`, `gamestate.episode`, `playstate` (ex_warped, ex_completed), scoring via `GivePoints()`
- **Rendering** (rt_draw.c, rt_vid.h): `ThreeDRefresh()`, screen setup/teardown, fade effects via `VL_FadeOut()`
- **UI/Menu system** (rt_menu.c, rt_menu.h): `CP_LevelSelectionMenu()`, `MU_JukeBoxMenu()`, `SetupMenuBuf()`, `SetUpControlPanel()`, menu navigation
- **Sound/Music** (rt_sound.c, rt_sound.h): `MU_StartSong()`, `MU_JukeBoxMenu()`, `MU_StoreSongPosition()`, `MU_RestoreSongPosition()`, `StopWind()`
- **Input management** (rt_in.h/isr.h): `Keyboard[]` array, `LetterQueue[]`, input shutdown/restore
- **Message system** (rt_msg.h): `AddMessage()` for cheat feedback (MSG_CHEAT type)
- **Demo system** (rt_main.h or similar): `SaveDemo()`, `LoadDemo()`, `DemoExists()` predicates
- **Map visualization** (rt_map.c): `CheatMap()` for full map reveal

## Design Patterns & Rationale

| Pattern | Implementation | Rationale |
|---------|---|---|
| **Command Pattern** | `Codes[]` lookup table → enum index → switch statement handler dispatch | Decouples cheat input (LetterQueue) from cheat effects (individual handlers). Enables easy addition of new cheats by extending enum and Codes table. |
| **Circular Ring Buffer** | `LetterQueue[MAXLETTERS]` with `(index & (MAXLETTERS-1))` wrapping | Fixed memory footprint; fast modulo via bitwise AND (works when MAXLETTERS is power-of-2). Avoids malloc for dynamic input history. |
| **State Machine** | `CheckDebug()` has multiple code paths based on `DebugOk`, `demorecord`, `demoplayback` flags | Controls which cheats are active in different game states. Prevents unintended cheat triggering during demo playback or when cheats disabled. |
| **Factory/Builder** | `CheatSpawnItem(int item)` → `SpawnStatic()` → `MakeStatActive()` on `LASTSTAT` | Encapsulates entity creation logic; ensures correct initialization (z-level, flags). Reused by multiple cheat handlers. |
| **Lazy Toggling** | Uses XOR operators (`^= 1`) for godmode, HUD, light diminishing toggles | Common 1990s idiom; avoids separate "on" and "off" functions. Single cheat code toggles state. |

**Why this structure?** The cheat system predates modern GUI-driven debug menus. Fixed lookup tables and enum dispatch were faster than runtime string parsing on 1990s CPUs. The LetterQueue approach works only for sequential keyboard codes (not key combos), reflecting the era's input capabilities.

## Data Flow Through This File

```
Input:  Keyboard[scancode] → isr.h/rt_in.h 
        ↓
LetterQueue[]: Ring buffer of recent letters typed (updated by extern code)
        ↓
CheckDebug() [called from main game loop, ~once per frame]:
  ├─ Gate 1: if not DebugOk, only check codes 0-1 (ENABLECHEAT variants)
  ├─ Gate 2: if demoplayback, skip all checks
  ├─ Gate 3: if demorecord, only check DEMORECORD/DEMOEND
  ├─ Gate 4: Otherwise, check all relevant codes via CheckCode()
        ↓
CheckCode(which):
  ├─ Scan LetterQueue backwards from tail
  ├─ Compare uppercase versions of Codes[which].code characters
  ├─ If match found: clear matched codes from queue, dispatch via switch
        ↓
Handler functions (DoGodMode, DoWarp, DoItemCheat, etc.):
  ├─ Modify player state (health, armor, keys, weapon, position)
  ├─ Spawn entities (powerups, armor, weapons)
  ├─ Trigger UI (menus, level selection)
  ├─ Call AddMessage(MSG_CHEAT) for HUD feedback
  └─ Return control to game loop
        ↓
Output: Player flags changed, entities spawned, menu state entered, music/sound triggered,
        game state modified (level warping), rendering updated
```

**Key state transitions:**
- `godmode ^= 1` → toggles between invulnerable/mortal
- `playstate = ex_warped` → signals game loop to load new level
- Demo record/playback flags set/cleared via handlers

## Learning Notes

**What this file teaches:**
1. **Input-to-action mapping in retro engines**: LetterQueue is a simple, efficient approach for single-keystroke-sequence detection. Modern engines use event-driven input and sceneography systems (Unreal's input action mappings, Unity's new Input System).
2. **Monolithic state mutation**: Unlike modern engines with state managers or Redux patterns, this code directly mutates globals (`player->health`, `godmode`, `gamestate.mapon`). Illustrates the cost of tight coupling.
3. **Cheat codes as first-class feature**: Cheats are baked into the release build (gated by `DebugOk` flag). Modern engines often strip cheat code logic in release builds entirely.
4. **Cross-cutting debug concerns**: Both cheat codes and frame-by-frame debug keys (`DebugKeys()`) reuse the same handler functions (e.g., `DoGodMode()`), creating a mixed abstraction. Shows how older codebases merged debug and gameplay concerns.
5. **Ring buffer elegance**: The `(index & MAXLETTERS-1)` pattern is a compact, O(1) circular buffer implementation still used in modern engines (audio ring buffers, network packet queues).

## Potential Issues

1. **Buffer overflow risk in LetterQueue scanning**: `CheckCode()` scans backwards through `LetterQueue` without visible bounds checking. If `MAXLETTERS` is misconfigured or LetterQueue corruption occurs, out-of-bounds read is possible.

2. **Demovideo replay conflict**: If a cheat handler is called during demo playback (which should be gated by `if (demoplayback) return;` in `CheckDebug()`), it can cause desync. The gate exists but is fragile — if a new code path bypasses it, desync occurs silently.

3. **Menu re-entrancy**: `DoWarp()` and `DoJukeBox()` call `SetUpControlPanel()` which may have state assumptions about being called from the main menu. Calling mid-game could cause undefined behavior if menu state machine isn't re-entrant.

4. **Networking state corruption**: Cheat handlers like `DoItemCheat()` that spawn entities and modify player state don't coordinate with the network subsystem (rt_net.c). In multiplayer, a cheat on one client could desync from server state.

5. **Hard-coded dual code variants**: Each cheat exists twice in the enum and Codes table (e.g., `ENABLECHEAT` and `ENABLECHEATALT`). Adds maintenance burden and table bloat; modern engines would use a single code with aliases.
