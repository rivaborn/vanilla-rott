# rott/rt_main.h — Enhanced Analysis

## Architectural Role

`rt_main.h` serves as the **central state interface** and **lifecycle manager** for the Rise of the Triad game engine. It declares the global `gamestate` structure that persists throughout a play session, defines the core update/initialization/shutdown callbacks that structure the main loop, and exports critical feature flags and timing globals. This header is foundational: nearly every game system that needs to know the current score, difficulty, remaining time, or player inventory must import this file.

## Key Cross-References

### Incoming (who depends on this file)
Based on the codebase structure, the following subsystems depend on `gametype` and lifecycle functions defined here:
- **Main game loop** (rt_main.c, engine.c) — calls `InitCharacter()`, `UpdateGameObjects()`, `PlayCinematic()`, `PauseLoop()`, `ShutDown()`
- **Battle system** (rt_battl.c) — reads/writes `gamestate.battlemode`, `gamestate.BattleOptions`, special timers
- **Player system** (rt_playr.c) — updates `gamestate` counts (kills, treasures, secrets), checks special power-up durations
- **Actor/enemy system** (rt_actor.c) — calls `UpdateGameObjects()` internally; spawns tied to game state
- **Menu system** (rt_menu.c) — reads difficulty, violence, score; modifies game state for level selection
- **UI/HUD** (rt_draw.c) — displays stats from `gamestate` (score, ammo, health, kill/treasure counts)
- **Network layer** (rt_net.c) — synchronizes `gamestate` across multiplayer clients
- **Save/load system** — persists `gamestate` to disk

### Outgoing (what this file depends on)
- **develop.h** — Debug/feature preprocessor symbols (DEBUG, WEAPONCHEAT, etc.)
- **rt_def.h** — Core constants: `MAXPLAYERS`, screen geometry, map/sprite limits, global flag definitions
- **rottnet.h** — Networking types: `MAXPLAYERS` constant for array bounds
- **rt_battl.h** — `battle_type` structure definition for multiplayer battle mode configuration

**Note:** The ARCHITECTURE_CONTEXT was truncated, but the CROSS-REFERENCE excerpt shows that functions like `CheckCommandLineParameters` (rt_main.c) and `CheckForQuickLoad` (_rt_main.h) are defined in implementation files tied to this header.

## Design Patterns & Rationale

1. **Monolithic Global State (gametype)**  
   - Single `extern gametype gamestate` holds all persistent game progress: scores, counts, difficulty, battle options, special power-up timers
   - Pro: Easy access from anywhere; minimal parameter passing
   - Con: Tight coupling; hard to isolate game logic; difficult to parallelize or snapshot state
   - *Rationale*: Common in 1990s engines; straightforward for single-player, extended to multiplayer via network sync

2. **Lifecycle Callback Functions**  
   - `SetupWads()` → `InitCharacter()` → main loop (`UpdateGameObjects()` each frame) → `ShutDown()` / `QuitGame()`
   - Decouples initialization, frame updates, and cleanup into distinct phases
   - *Rationale*: Mirrors the classic game loop pattern; allows each subsystem to initialize/update/shutdown in order

3. **Feature Flags for Testing (SCREENSHOTS, MEMORYTEST, MODEMTEST, etc.)**  
   - Boolean flags can be set at compile time or via command line to enable special test modes
   - *Rationale*: Development-era practice to enable profiling, memory validation, and network testing without code recompilation

4. **Version Enum (version_type)**  
   - Game variant branches: ROTT_SHAREWARE, ROTT_REGISTERED, ROTT_SUPERCD, ROTT_SITELICENSE
   - Allows runtime content/feature gating (e.g., episode limits, online features)
   - *Rationale*: Publishing model of the era (shareware with feature tiers)

## Data Flow Through This File

```
Startup Phase:
  main() → SetupWads() → InitCharacter()
           (load WAD data)    (initialize player, gamestate)

Main Loop (per frame):
  Main game loop → UpdateGameObjects()
                → reads/writes gamestate (score, kills, power-up timers)
                → updates actor state

Pause/Menu:
  Input → PauseLoop()
       → menus read/write gamestate (e.g., difficulty, next level)

End of Game:
  Quit triggered → ShutDown()
               → cleanup resources
               → QuitGame()
               → exit

Power-Up Mechanics:
  collectible picked up → special.GodModeTime++ (via player code)
  each frame in UpdateGameObjects() → special timers decrement
  if timer expires → power-up effect ends
```

**Key State Mutations:**
- `gamestate.killcount`, `treasurecount`, `secretcount` — incremented when player kills enemy or collects item
- `gamestate.score` — increased on kills/items; sent to HUD for display
- `gamestate.SpecialsTimes.*` — countdown timers; checked by player code to determine active effects
- `gamestate.battlemode`, `BattleOptions` — set in menu; read during multiplayer match startup
- `gamestate.randomseed` — consumed by level generation/enemy AI for deterministic randomness

## Learning Notes

### Engine Idioms
- **Monolithic state struct**: Represents a "single entity" game model (one player, one game progress). Modern engines often split this into multiple subsystems (Health component, Score component, etc.) or use ECS.
- **Global extern pattern**: All subsystems import and directly access `extern gametype gamestate`. Modern practice favors dependency injection or event systems to reduce coupling.
- **Lifecycle callbacks**: `SetupWads()`, `InitCharacter()`, `UpdateGameObjects()`, `ShutDown()` mirror the **Three-Phase Initialization pattern**: init → update loop → cleanup.
- **Respawn Timers in Struct**: `specials` stores both active duration (`GodModeTime`) and respawn delay (`GodModeRespawnTime`), enabling both gameplay mechanics and item recycling logic.

### Era-Specific Design
- **Hardcoded Power-Up Limits**: Fixed array of 8 special types suggests design-time decision (power-ups were hand-authored, not data-driven).
- **Version Gating**: Enum-based feature tiers reflect commercial publishing constraints of 1995.
- **Feature Flags (SCREENSHOTS, MEMORYTEST, MODEMTEST)**: Reflect the need for embedded profiling and debug modes on constrained hardware (DOS era).

### Cross-System Dependencies
- The `battle_type` field shows tight coupling to multiplayer subsystem; battle state is co-located with single-player state rather than in a separate battle manager.
- The `MAXPLAYERS` include from rottnet.h indicates that player arrays throughout the codebase are sized to this constant, defined in the networking layer—architectural decision that networking was designed first.

## Potential Issues

1. **Tight Coupling via Global State**  
   Any subsystem can mutate `gamestate` at any time, making it hard to reason about state mutations or debug unexpected changes. Modern engines use event buses or state machines to make flows explicit.

2. **Power-Up Timer Management**  
   `specials` struct stores raw int durations (frames? milliseconds?). No apparent delta-time tracking, suggesting frame-rate-dependent timing—could cause bugs if frame rate is unstable or in networked games with frame skips.

3. **Array Sizing by Constant**  
   `PlayerHasGun[ MAXPLAYERS ]` and similar arrays assume `MAXPLAYERS` is known at compile time. If multiplay limits ever change, all client/server binaries must be recompiled in sync—a fragility risk for patching.

4. **Multiplayer Synchronization Complexity**  
   With 30+ fields in `gametype`, any out-of-sync field can cause desync bugs. No apparent versioning or CRC check in the structure definition, so protocol changes are risky.
