# rott/rt_battl.h

## File Purpose
Public header for the battle system (multiplayer modes) in Rise of the Triad. Defines battle mode types, configuration options, event codes, and declarations for initializing and managing multiplayer/deathmatch gameplay.

## Core Responsibilities
- Define battle modes (Normal, Collector, Scavenger, Hunter, Tag, Eluder, Deluder, Capture The Triad, etc.)
- Declare battle event types and status return codes
- Define battle configuration options (speed, ammo, hit points, light levels, kill limits, damage)
- Expose battle system state variables (kill tracking, player points, team assignments)
- Declare core battle lifecycle functions (init, shutdown, event handling)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `battle_status` | enum | Return codes: `battle_no_event`, `battle_end_game`, `battle_end_round`, `battle_out_of_time` |
| `battle_event` | enum | Event types: player killed, tagged, kill methods (missile/bullet/air/crush), collector item, eluder caught, etc. |
| Battle modes | enum | `battle_StandAloneGame`, `battle_Normal`, `battle_ScoreMore`, `battle_Collector`, `battle_Scavenger`, `battle_Hunter`, `battle_Tag`, `battle_Eluder`, `battle_Deluder`, `battle_CaptureTheTriad` |
| Speed/Ammo/Light/Kills enums | enum | Game option configurations (normal/fast speed, ammo types, light levels, kill targets) |
| `battle_type` | struct | Complete battle mode configuration: Gravity, Speed, Ammo, HitPoints, SpawnDangers, SpawnHealth, SpawnWeapons, SpawnMines, RespawnItems, WeaponPersistence, RandomWeapons, FriendlyFire, LightLevel, Kills, DangerDamage, TimeLimit, RespawnTime |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `BATTLEMODE` | boolean | extern | Battle mode active flag |
| `WhoKilledWho` | short[MAXPLAYERS][MAXPLAYERS] | extern | Kill matrix tracking who killed whom |
| `BATTLE_Points` | short[MAXPLAYERS] | extern | Player score/points |
| `BATTLE_PlayerOrder` | short[MAXPLAYERS] | extern | Ranked player ordering |
| `BATTLE_NumCollectorItems` | int | extern | Collector mode item count |
| `BATTLE_It` | int | extern | Current "It" player for tag modes |
| `BATTLE_Team` | short[MAXPLAYERS] | extern | Team assignment per player |
| `BATTLE_TeamLeader` | short[MAXPLAYERS] | extern | Team leader per player |
| `BATTLE_NumberOfTeams` | int | extern | Active team count |
| `BATTLE_Options` | battle_type[battle_NumBattleModes] | extern | Pre-configured options per mode (in RT_MENU.C) |

## Key Functions / Methods

### BATTLE_Init
- Signature: `void BATTLE_Init(int battlemode, int numplayers)`
- Purpose: Initialize battle system for specified mode with given player count
- Inputs: Battle mode enum, player count
- Outputs/Return: None
- Side effects: Sets up global battle state, likely resets kill/point tracking

### BATTLE_CheckGameStatus
- Signature: `battle_status BATTLE_CheckGameStatus(battle_event reason, int player)`
- Purpose: Poll game status and determine if game should end based on battle event
- Inputs: Event type (kill, tag, etc.), player index
- Outputs/Return: `battle_status` (no_event, end_game, end_round, out_of_time)
- Side effects: May update points, check win conditions, manage round/game end

### BATTLE_PlayerKilledPlayer
- Signature: `battle_status BATTLE_PlayerKilledPlayer(battle_event reason, int killer, int victim)`
- Purpose: Handle kill event; update kill matrix and scores
- Inputs: Kill type (missile/bullet/crush/etc.), killer and victim player indices
- Outputs/Return: `battle_status` (game continuation or end status)
- Side effects: Updates `WhoKilledWho`, `BATTLE_Points`, kill tracking

### BATTLE_SetOptions / BATTLE_GetOptions
- Signature: `void BATTLE_SetOptions(battle_type *options)`, `void BATTLE_GetOptions(battle_type *options)`
- Purpose: Configure or retrieve current battle mode options
- Inputs/Outputs: Pointer to battle_type configuration struct

### BATTLE_SortPlayerRanks
- Signature: `void BATTLE_SortPlayerRanks(void)`
- Purpose: Re-order players by score/points, update `BATTLE_PlayerOrder`
- Side effects: Modifies `BATTLE_PlayerOrder` array

### BATTLE_Shutdown
- Signature: `void BATTLE_Shutdown(void)`
- Purpose: Clean up battle system on mode exit

## Notes
- Trivial helpers: `BIT_MASK` and `MINUTES_TO_GAMECOUNT` macros for bit operations and time conversion
- Battle configuration includes spawn settings (dangers, health, weapons, mines), weapon persistence, friendly fire toggle, and respawn timing
- Kill targets can be random (`-2`), blind (`-1`), infinite (`0`), or default (`21`)
- Danger damage can be normal (`-1`), low (`1`), or instant kill (`30000`)
- Time limits use `0` to signify infinite
- Default respawn time is 30 seconds

## Control Flow Notes
Not inferable from this header alone. This is a configuration/state interface; actual battle loop logic resides in RT_BATTL.C.

## External Dependencies
- Assumes `MAXPLAYERS` constant defined elsewhere
- Assumes `VBLCOUNTER` defined (time reference)
- References RT_MENU.C for `BATTLE_Options` array initialization
