# rott/rt_game.c — Enhanced Analysis

## Architectural Role

**rt_game.c** serves as the game loop's **HUD/UI facade and progression controller**, bridging the 3D engine (engine.c, rt_draw.c) with input, audio, and state persistence subsystems. It executes the game state machine—level progression, death/respawn, save/load cycles—while rendering a multi-modal status display that adapts to single-player campaign, battle modes, and demo playback. Its pervasive use of `SHOW_TOP_STATUS_BAR()`, `SHOW_BOTTOM_STATUS_BAR()`, and `BATTLEMODE` macros reveals architectural dependency inversion: the HUD doesn't know *why* it's hidden or what mode is active; it queries compile-time and runtime predicates to determine what to render.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_main.c** (game loop): Calls `SetupPlayScreen()` at level init; calls `DrawPlayScreen()` every frame
- **rt_menu.h** (control panel): Calls `CheckHighScore()`, `SaveTheGame()`, `LoadTheGame()` for save/load UI
- **rt_playr.c** (player controller): Calls `Died()` on player death; updates `locplayerstate->poweruptime` which rt_game.c renders
- **rt_actor.c** (enemy/object spawning): Calls `SpawnPlayerobj()` (actor system integration) when respawning
- **rt_net.c** (multiplayer): Calls `BattleLevelCompleted()` at end of multiplayer rounds; rt_game.c reads `PLAYERSTATE[]`, `BATTLE_Points[]`, `WhoKilledWho[]`

### Outgoing (what this file depends on)
- **WAD/Resource**: `W_CacheLumpName()`, `W_GetNumForName()`, `W_CacheLumpNum()` → w_wad.c
- **Video/Rendering**: `VL_MemToScreen()`, `GM_MemToScreen()`, `VGAMAPMASK()`, `VL_FadeOut()`, `VL_FadeIn()` → rt_vid.c, modexlib
- **Audio**: `SD_Play()`, `MU_StartSong()`, `MU_SaveMusic()`, `MU_LoadMusic()` → rt_sound.c, isr.c
- **Game Loop**: `UpdateGameObjects()`, `ThreeDRefresh()`, `CalcTics()` → rt_main.c, engine.c
- **Player System**: `SpawnPlayerobj()`, `LoadPlayer()`, `ResetPlayerstate()` → rt_playr.c
- **World**: `ConnectAreas()`, `LoadDoors()`, `LoadElevators()` → rt_door.c (save/load restore)
- **Battle System**: Reads `BATTLE_Points[]`, `WhoKilledWho[]`, calls `BATTLE_SortPlayerRanks()` → rt_battl.c

## Design Patterns & Rationale

**Lazy Caching (SetupPlayScreen)**  
All HUD graphics are cached once per level with `CacheLumpGroup()`, bypassing per-frame WAD lookups. Rationale: 1995 hardware; disk I/O was the bottleneck. Assets are tagged `PU_LEVEL` so they auto-free on level transition.

**Planar VGA Rendering (DrawMPPic)**  
Custom implementation of 4-plane VGA sprite rendering with per-pixel masking (255 = skip). Rather than generic `VWB_DrawPic()`, rt_game.c implements `DrawMPPic()` to support partial-height rendering via `heightmod`—used to animate health bar fill and powerup bar descent. This is a **performance micro-optimization for DOS VGA**: one plane per CPU loop avoids bank-switching penalty.

**Mode-Conditional Rendering**  
The `SHOW_TOP_STATUS_BAR()`, `SHOW_KILLS()`, `BATTLEMODE` macro gates suggest **compile-time and runtime mode abstraction**. The caller (rt_main.c) doesn't call separate `DrawPlayScreenSinglePlayer()` vs `DrawPlayScreenBattle()`; instead, rt_game.c branches internally. Rationale: single entry point, reduced code duplication.

**Tag-Based Save Format**  
`SaveTheGame()` writes sequential tag+data pairs (ROTT header, DOOR, ELEVATOR, etc.) rather than fixed-offset struct serialization. This allows **version-resilient save loading**: older saves missing a tag skip it; new saves with extra tags are ignored by old code. Implies the engine evolved through shipped versions with save-file forward/backward compatibility.

**Bifurcated Death Paths**  
`Died()` chooses between **cinematic zoom-out** (if `ZoomDeathOkay()` true), **rotation to face killer**, or **instant fade**. Suggests different game pacing expectations: campaign mode favors dramatic death; multiplayer favors snappy respawn. The "dopefish" mode (`flags & FL_DOPEFISH`) spawns a dummy body with chaotic physics—likely a cheat/dev mode Easter egg.

## Data Flow Through This File

**Per-Level Initialization:**
```
Level Load → SetupPlayScreen() {
  Cache 40+ pic_t from WAD (digits, icons, health, ammo, men, keys)
  Store in static pic_t* arrays (lifeptnums[], ammo[], etc.)
  Initialize oldplayerhealth = -1 (force first draw)
}
```

**Per-Frame Rendering:**
```
DrawPlayScreen() {
  Read gamestate.score, locplayerstate->lives, player->flags
  Conditionally render: top bar (time, player name, lives, score, keys, powerups)
  Render bottom bar (ammo, health) or kills (battle mode)
  Cache sprite on-demand (W_CacheLumpName "stat_bar", "bottbar", etc. with PU_CACHE)
  Call DrawTime(), DrawBarHealth(), DrawBarAmmo(), DrawKills()
  Apply screen shake offset to displayofs
}
```

**Death Sequence:**
```
Player.health <= 0 → Died() {
  Localize attacker from player->ticcmd
  Spawn dummy actor copy (for cinematic)
  Rotate view to face attacker / zoom out
  Decrement lives
  Respawn or GameOver
}
```

**Level Completion:**
```
Exit trigger → LevelCompleted(exit_t) {
  Calculate 11 bonus categories (time left, kills, secrets, etc.)
  Animate score +bonus with sound cues
  If all bonuses: loop back to final "BONUS BONUS" screen
  Play music, fade in next level
}
```

**Save/Load:**
```
SaveTheGame(slot, gamestate) → Write WAD-like tagged chunks (GAMESTATE, DOOR, ACTOR, etc.)
LoadTheGame(slot, gamestate) → Read, validate CRC, rebuild world, restore player pos/ammo
```

## Learning Notes

1. **Era-Specific VGA Optimizations**: `DrawMPPic()` directly manipulates `VGAMAPMASK()` register and processes planes sequentially. Modern engines batch-render sprites; 1995 DOOM-era required low-level driver knowledge.

2. **Lazy Resource Loading**: `SetupPlayScreen()` defers all caching to level start, not engine startup. Reflects cartridge/ROM memory limits of the era; CD-based games could prefetch globally.

3. **Macros as Abstraction**: `SHOW_TOP_STATUS_BAR()`, `BATTLEMODE` are compile-time/runtime switches. No virtual functions or inheritance; C macros and runtime branches instead.

4. **Stateful UI Dirtyness**: `oldplayerhealth`, `oldpercenthealth`, `oldsec` track previous frame state to avoid redrawing unchanged HUD elements. Modern GPUs render every frame; this code avoids pixel writes.

5. **Asymmetric Death Handling**: Single-player death is cinematic (zoom/rotate); multiplayer death is snappy. Suggests design-for-feel rather than unified system.

6. **Tag-Based Persistence**: Save format uses `ROTT` header + tags, not fixed structs. Forward-compatible. Likely influenced by Doom's approach or custom evolution.

## Potential Issues

1. **Global Mutable State**: `SHAKETICS`, `damagecount`, `poweruptime`, `oldplayerhealth` are static globals modified by this file and read by `rt_draw.c` (ScreenShake) / `rt_main.c`. Race conditions possible if multithreaded; suggests single-threaded game loop assumption.

2. **Hardcoded Limits**: `MaxScores` (high score table), `MAXKILLBOXES` (10), `numplayers` assumed ≤ 5. No dynamic allocation. Multiplayer scaling beyond 5 players would break rankings display.

3. **Unchecked CRC Recovery**: `LoadTheGame()` has a "corrupted" recovery path that may load partially-broken saves. Could lead to silent data loss if CRC fails but load proceeds.

4. **Death Camera Rotation Fragility**: The cinematic death zoom uses `RotateBuffer()` in a loop with frame-by-frame updates. If `ThreeDRefresh()` has latency variance, animation may stutter.

5. **WAD Asset Lifecycle**: Assets cached in `SetupPlayScreen()` are tagged `PU_LEVEL` but some (like ammo/health) are generic. If two levels load different ammo graphics with the same lump name, cache collision could occur.

---

**Generated with architectural cross-reference analysis** | Context: Game loop HUD facade, multiplayer battle mode integration, save/load subsystem
