# rott/rt_battl.c

## File Purpose
Implements battle/multiplayer game mode support for Rise of the Triad, including mode initialization, round management, kill tracking, score calculation, and game-state transitions based on battle events. Manages team assignments, point goals, and special game rules for deathmatch variants.

## Core Responsibilities
- Initialize battle system with selected mode (Normal, Tag, Hunter, Collector, etc.) and player configuration
- Track and update player/team points, kills, and rankings across rounds
- Handle game state transitions triggered by battle events (kills, item collection, time limits)
- Validate battle mode rules and enforce friendly-fire/spawn settings
- Sort player rankings and synchronize score displays
- Calculate points awarded based on kill type and battle mode
- Manage round lifecycle (start, refresh timer, end conditions)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `battle_status` | enum | Return codes for game status checks: `battle_no_event`, `battle_end_game`, `battle_end_round`, `battle_out_of_time` |
| `battle_event` | enum | Event types triggering status checks: player killed, item collected, eluder caught, etc. |
| `battle_type` | struct | Battle configuration: gravity, speed, ammo, hitpoints, spawn flags, respawn time, friendly fire, kill goal, damage |
| `specials` | struct | Timing durations for special power-ups (god mode, shrooms, etc.) and their respawn intervals |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Timer` | int | static | Round timer (incremented each refresh) |
| `TimeLimit` | int | static | Maximum ticks before round ends (INFINITE if unlimited) |
| `NumberOfPlayers` | int | static | Active player count in battle |
| `BattleRound` | int | static | Current round number (incremented per restart) |
| `BattleMode` | int | static | Current battle mode enum value |
| `RoundOver` | boolean | static | Flag indicating round should end |
| `KillsEndGame` | boolean | static | Whether reaching kill goal ends game |
| `KeepTrackOfKills` | boolean | static | Whether to update scores on kill events |
| `UpdateKills` | boolean | global | Flag to trigger score display redraw |
| `SwapFlag` | boolean | static | Set during sorting if any swaps occurred |
| `BattleOptions` | battle_type | static | Copy of battle configuration for this match |
| `BattleSpecialsTimes` | specials | static | Hardcoded special item durations (god: 60 tics, respawn: 300 tics, etc.) |
| `BATTLEMODE` | boolean | global | True if in multiplayer battle mode (vs. standalone game) |
| `WhoKilledWho` | short[MAXPLAYERS][MAXPLAYERS] | global | Kill matrix: `[killer][victim]` counts |
| `BATTLE_Points` | short[MAXPLAYERS] | global | Team/player score array |
| `BATTLE_PlayerOrder` | short[MAXPLAYERS] | global | Player indices sorted by rank |
| `BATTLE_Team` | short[MAXPLAYERS] | global | Team assignment per player |
| `BATTLE_TeamLeader` | short[MAXPLAYERS] | global | Leader player index per team |
| `BATTLE_NumberOfTeams` | int | global | Active team count |
| `BATTLE_NumCollectorItems` | int | global | Remaining items to collect in Collector mode |
| `PointGoal` | int | global | Target score to win (or kills needed) |
| `DisplayPoints` | int | global | UI display copy of point goal |
| `BATTLE_It` | int | global | Current "it" player in Tag/Hunter modes |

## Key Functions / Methods

### BATTLE_Init
- **Signature:** `void BATTLE_Init(int battlemode, int numplayers)`
- **Purpose:** Initialize all battle system state for a new match; set default game options and mode-specific rules.
- **Inputs:** `battlemode` (enum value for game mode), `numplayers` (1 to MAXPLAYERS)
- **Outputs/Return:** None (modifies global state)
- **Side effects:** 
  - Resets Timer, RoundOver, BattleRound to initial values
  - Initializes all score arrays and team assignments
  - Calls `BATTLE_StartRound()` to begin first round
  - Modifies `gamestate.BattleOptions` based on selected mode and `BattleOptions` config
  - Sets spawn flags (health, weapons, dangers, collector items) per mode
  - Updates global `GRAVITY` based on options
- **Calls:** `GameRandomNumber()` (for random kill goals), `BATTLE_StartRound()`, `Error()`, `SoftError()` (debug)
- **Notes:** 
  - Enforces mode restrictions: Tag only in free-for-all, Capture the Triad only in team mode
  - Shareware version limited to Normal/Collector/Hunter modes
  - Hunter mode initializes round-robin "it" rotation

### BATTLE_StartRound
- **Signature:** `static battle_status BATTLE_StartRound(void)`
- **Purpose:** Begin a new round; set up round timer and determine "it" player for Hunter mode.
- **Inputs:** None (uses global `BattleMode`, `BATTLEMODE`, `BattleOptions`)
- **Outputs/Return:** `battle_status` enum (`battle_no_event` or `battle_end_game`)
- **Side effects:**
  - Increments `BattleRound`
  - Resets `Timer` and `RoundOver`
  - Converts time limit options to tick counts via `MINUTES_TO_GAMECOUNT()` macro
  - In Hunter mode: enables guns for all players, then disables for current "it" team
  - May return `battle_end_game` if Hunter mode PointGoal reached
- **Calls:** None directly
- **Notes:** Returns early with `battle_no_event` if not in BATTLEMODE

### BATTLE_CheckGameStatus
- **Signature:** `battle_status BATTLE_CheckGameStatus(battle_event reason, int player)`
- **Purpose:** Process game events (refresh, kills, pickups) and update battle state; determine if game should end.
- **Inputs:** 
  - `reason` (battle_event enum: refresh, player_killed, collector_item, etc.)
  - `player` (player index, validated 0 to MAXPLAYERS-1)
- **Outputs/Return:** `battle_status` (no_event, end_game, end_round, out_of_time)
- **Side effects:**
  - Increments timer on `battle_refresh`
  - Updates `BATTLE_Points` based on event type and `BattleMode`
  - Sets `RoundOver` and calls score display (`DrawKills`) if time limit exceeded
  - Calls `BATTLE_StartRound()` in Hunter mode when time limit expires
  - May call `RespawnEluder()`, `SpawnCollector()`, `AddMessage()`
  - Respawns Eluder/Deluder NPCs after caught/shot
- **Calls:** `BATTLE_SortPlayerRanks()`, `DrawKills()`, `SD_Play()`, `AddMessage()`, `BATTLE_StartRound()`, `RespawnEluder()`, `Error()` (debug)
- **Notes:** 
  - Mode-specific behavior: Normal/ScoreMore award points on kill; Tag and Hunter have special rules
  - Friendly-fire toggle controls point penalties for team kills
  - Eluder/Deluder modes end round when point goal reached

### BATTLE_PlayerKilledPlayer
- **Signature:** `battle_status BATTLE_PlayerKilledPlayer(battle_event reason, int killer, int victim)`
- **Purpose:** Award points for a kill event; apply mode-specific scoring rules and check for game end.
- **Inputs:** 
  - `reason` (kill method: missile, bullet, in-air, crushing, player-tagged)
  - `killer`, `victim` (player indices, both 0 to MAXPLAYERS-1)
- **Outputs/Return:** `battle_status` (no_event or end_game)
- **Side effects:**
  - Updates `WhoKilledWho[killer][victim]` kill matrix
  - Modifies `BATTLE_Points[killerteam]` based on reason and mode:
    - ScoreMore: 1–4 points depending on kill type
    - Tag: tags increment victim's team score; points go to tagger's team
    - Hunter: only counts kills on current "it" team; penalties for team kills
    - Normal/others: 1 point per kill (minus for friendly fire)
  - Sets `UpdateKills = true` to trigger score refresh
  - Sets `RoundOver = true` and returns `battle_end_game` if point goal reached
  - Calls `AddMessage()` for crushing kills
- **Calls:** `AddMessage()`, `Error()` (debug validation)
- **Notes:** 
  - Validates killer/victim indices and teams are in valid ranges
  - Self-kill only allowed with missile/in-air reasons
  - Friendly-fire penalty only applied if `BattleOptions.FriendlyFire` is true

### BATTLE_SortPlayerRanks
- **Signature:** `void BATTLE_SortPlayerRanks(void)`
- **Purpose:** Bubble-sort players by score and update `BATTLE_PlayerOrder` ranking; update "it" player in non-Tag modes.
- **Inputs:** None (uses `BATTLE_Points[]`, `BATTLE_PlayerOrder[]`)
- **Outputs/Return:** None (modifies `BATTLE_PlayerOrder[]` and `BATTLE_It`)
- **Side effects:**
  - Reorders `BATTLE_PlayerOrder` by score (ascending for Tag, descending for others)
  - Sets `BATTLE_It = BATTLE_PlayerOrder[0]` (top scorer) in non-Hunter modes
  - Sets `SwapFlag = true` if any swaps occurred
  - Plays sound `SD_ENDBONUS1SND` if scores changed and scores are visible
- **Calls:** `SD_Play()`, `SoftError()` (debug)
- **Notes:** Simple O(n²) bubble sort; compares `BATTLE_Points[BATTLE_PlayerOrder[i]]`

### BATTLE_GetSpecials
- **Signature:** `void BATTLE_GetSpecials(void)`
- **Purpose:** Copy hardcoded special item durations from `BattleSpecialsTimes` into `gamestate.SpecialsTimes`, scaling by VBLCOUNTER.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Overwrites `gamestate.SpecialsTimes` array with scaled durations
- **Calls:** None
- **Notes:** Uses integer pointer arithmetic to iterate; copies 16 entries (8 special types + 8 respawn times)

### BATTLE_SetOptions / BATTLE_GetOptions
- **Signature:** 
  - `void BATTLE_SetOptions(battle_type *options)`
  - `void BATTLE_GetOptions(battle_type *options)`
- **Purpose:** Store/retrieve battle configuration from/to module static storage.
- **Inputs/Outputs:** `battle_type *options` pointer
- **Side effects:** `memcpy()` to/from `BattleOptions` static
- **Calls:** `memcpy()`
- **Notes:** Simple wrappers; no validation

### BATTLE_Shutdown
- **Signature:** `void BATTLE_Shutdown(void)`
- **Purpose:** Reset all battle state to defaults (single-player standalone game).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** 
  - Clears `Timer`, `BattleRound`, `BattleMode`, `BATTLEMODE`, player counts
  - Zeros all scores, kill matrix, team assignments
  - Resets spawn flags to defaults (health/weapons/dangers enabled, no specials)
  - Resets `gamestate` fields to standalone defaults
- **Calls:** None
- **Notes:** Called on game shutdown or battle-to-singleplayer transition

## Control Flow Notes
- **Initialization path:** `BATTLE_Init()` → `BATTLE_StartRound()` (called once at match start)
- **Per-frame update:** `BATTLE_CheckGameStatus(battle_refresh, player)` triggered by game main loop
- **Event handling:** Other `BATTLE_CheckGameStatus()` calls via external modules when players kill, collect items, etc.
- **Scoring:** `BATTLE_PlayerKilledPlayer()` called by actor/damage system; updates points and may trigger `BATTLE_StartRound()` in Hunter mode
- **Display update:** `BATTLE_SortPlayerRanks()` called on `UpdateKills` flag; triggers `DrawKills()` if scores visible
- **End game:** When point goal or time limit hit, `RoundOver = true` and returns `battle_end_game`

## External Dependencies
- **Global state:** `gamestate` (BattleOptions, ShowScores, SpawnHealth, SpawnWeapons, SpawnDangers, SpawnCollectItems, Product, SpecialsTimes); `PLAYERSTATE[]` (player colors/uniforms); `consoleplayer` (current player index); `GRAVITY`
- **Functions defined elsewhere:**
  - `GameRandomNumber()` (rt_rand.c) – random number generation
  - `SD_Play()` (rt_sound.c) – play sound effect
  - `DrawKills()` (rt_view.c) – render score display
  - `SHOW_TOP_STATUS_BAR()`, `SHOW_KILLS()` (rt_view.c) – UI visibility checks
  - `AddMessage()` (rt_msg.c) – game message queue
  - `RespawnEluder()`, `SpawnCollector()` (rt_actor.c) – NPC spawn/respawn
  - `Error()`, `SoftError()` (debugging macros, conditional on BATTLECHECK/BATTLEINFO flags)
  - `MINUTES_TO_GAMECOUNT()` (macro in rt_battl.h) – time conversion
- **Notable includes:** rt_def.h (constants, types), rottnet.h (networking), isr.h (timer constants), rt_battl.h (public interface), rt_actor.h, rt_rand.h, rt_playr.h, rt_game.h, rt_sound.h, rt_com.h, rt_msg.h, rt_view.h, rt_util.h, rt_main.h, memcheck.h
