# rott/rt_playr.h

## File Purpose
Header file defining player state, movement, input handling, and weapon systems for the ROTT game engine. Declares interfaces for player spawning, control polling, combat mechanics, and multiplayer support.

## Core Responsibilities
- Define player state structure (health, weapons, position, input state)
- Declare player input polling functions (keyboard, mouse, joystick, special devices)
- Manage player movement, collision, and physics
- Control weapon selection, firing, and item acquisition
- Define attack sequences and weapon configuration data
- Support character selection and statistics
- Interface with network multiplayer system
- Track dead players and respawn mechanics

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `playertype` | struct | Primary player state: health, weapons, ammo, position, momentum, input buttons, animation state, target info |
| `ROTTCHARS` | struct | Character statistics: top speed, hit points, accuracy, height |
| `williamdidthis` | struct | Weapon configuration: base damage, impulse, attack frames/actions, ammo info |
| `attack_t` | struct | Attack action type and animation frame data |
| `missile_stats` | struct | Missile weapon properties: state, speed, class, offset, flags |
| `SWIFT_3DStatus` | struct | 3D input device status: X/Y/Z position, pitch/roll/yaw, buttons |
| `SWIFT_StaticData` | struct | 3D input device hardware info: type, version, coordinate descriptor |
| `attack_action` | enum | Attack state machine: reset, knife, trigger, automatic, dryheaving, missile |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PLAYERSTATE[]` | playertype | global | Player state array (one per player) |
| `PLAYER[]` | objtype* | global | Player object pointers (game objects) |
| `player` | objtype* | global | Current/active player pointer |
| `WEAPONS[]` | williamdidthis | global | Weapon configuration table (indexed by weapon type) |
| `PlayerMissileData[]` | missile_stats | global | Missile weapon properties table |
| `GRAVITY` | int | global | Current gravity value (LOW/NORMAL/HIGH) |
| `DEADPLAYER[]` | statobj_t* | global | Array of dead player objects |
| `NUMDEAD` | int | global | Count of dead players (max 32) |
| `buttonpoll[]` | boolean | global | Input button poll state for all buttons |
| `cybermanenabled`, `mouseenabled`, `joystickenabled` | boolean | global | Input device enable flags |
| `godmode`, `missilecam` | boolean | global | Debug/camera mode flags |
| `characters[]` | ROTTCHARS | global | Character stats table (5 characters) |

## Key Functions / Methods

### PollControls
- **Purpose**: Master control polling function; aggregates all input device polls
- **Inputs**: None
- **Outputs/Return**: None (updates global button/control state)
- **Side effects**: Updates `buttonstate[]`, `buttonheld[]`, `controlupdatetime`; calls network/demo functions
- **Calls**: PollKeyboardButtons, PollMouseButtons, PollJoystickButtons, PollCyberman; AddDemoCmd, AddRemoteCmd
- **Notes**: Central input aggregation point; called once per game frame

### PlayerMove
- **Purpose**: Main player movement logic; integrates input, momentum, collision, and physics
- **Inputs**: `objtype *ob` - player object
- **Outputs/Return**: None (modifies `ob` position, momentum, state)
- **Side effects**: Updates object position, momentum, falling state; calls collision functions
- **Calls**: ClipPlayer, PlayerSlideMove, ActorMovement, CheckPlayerSpecials
- **Notes**: Integrates gravity, momentum, slide movement; called once per frame per player

### Cmd_Fire
- **Purpose**: Execute firing command based on current weapon
- **Inputs**: `objtype *ob` - player object
- **Outputs/Return**: None (triggers weapon attack sequence)
- **Side effects**: Modifies weapon animation state, creates projectiles/effects, plays sounds
- **Calls**: Spawns missiles or bullets; calls weapon-specific effects
- **Notes**: Respects ammo, weapon state, and attack animation frame

### GetBonus
- **Purpose**: Award item/pickup to player; update stats and inventory
- **Inputs**: `objtype *ob` - player object; `statobj_t *stat` - item object
- **Outputs/Return**: None (modifies player stats/inventory)
- **Side effects**: Updates health, ammo, weapons, powerups; plays sound; flags item for removal
- **Calls**: Related pickup/award functions
- **Notes**: Central bonus distribution point

### SpawnPlayerobj
- **Purpose**: Create and initialize player object in the game world
- **Inputs**: `int x, y, z, angle` - spawn position and direction
- **Outputs/Return**: None (creates global PLAYER object)
- **Side effects**: Allocates and initializes player object; adds to actor list; resets player state
- **Calls**: GetNewActor, ResetPlayerstate, InitializeWeapons
- **Notes**: Called at game start and respawn

## Control Flow Notes
- **Init**: `SpawnPlayerobj`, `InitializeWeapons`, `ResetPlayerstate` set up initial state
- **Frame Loop**: `PollControls` reads input → `PlayerMove` updates physics → `UpdatePlayers` processes all players → rendering uses position
- **Combat**: `Cmd_Fire` and `Cmd_Use` triggered by polled input; create effects/projectiles
- **Shutdown**: `LoadPlayer` / `SaveWeapons` save/restore state for multiplayer sync

## External Dependencies
- **Includes**: rt_actor.h (objtype, classtype), rt_stat.h (statobj_t), states.h (statetype), rottnet.h (multiplayer), rt_battl.h (battle mode), develop.h (build flags)
- **Defined elsewhere**: objtype, statetype, statobj_t, classtype, thingtype, exit_t, dirtype (from included headers)
- **Notable globals**: BATTLEMODE (battle mode flag), firstactive/lastactive (actor list pointers), angletodir[] (angle lookup)
