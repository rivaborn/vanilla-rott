# rott/_rt_acto.h

## File Purpose
Private header for the actor/character system defining constants, macros, and data structures for managing game actors (enemies, projectiles, props). Implements state machines, physics parameters, and collision/behavior utilities for Rise of the Triad.

## Core Responsibilities
- Define actor state machine (11 states: STAND, PATH, COLLIDE, CHASE, AIM, DIE, FIRE, WAIT, CRUSH, etc.)
- Provide physics constants (speeds, momentum scales, fall/rise clipping planes)
- Define actor behavioral flags (PLEADING, EYEBALL, UNDEAD, attack modes)
- Implement inline macros for collision checks, direction parsing, state transitions
- Define saved actor structures for serialization
- Prototype core actor behavior functions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `saved_actor_type` | struct | Serializable actor state snapshot (position, health, angle, momentum, state index) |
| `tpoint` | struct | 3D point location with tile coordinates and fixed-point xyz |
| Actor states enum | enum | 11 discrete states (STAND, PATH, COLLIDE1/2, CHASE, USE, AIM, DIE, FIRE, WAIT, CRUSH) |

## Global / File-Static State
None defined here. File references external globals:
- `MISCVARS` (boss sound/visual timers)
- `UPDATE_STATES` array (state table lookup)
- `gamestate` (violence settings)
- `doorobjlist` (door objects)
- `BAS` array (base actor stats)

## Key Functions / Methods

### MissileMovement
- **Signature:** `void MissileMovement(objtype*)`
- **Purpose:** Update projectile physics each frame
- **Inputs:** Actor object pointer
- **Outputs/Return:** None
- **Side effects:** Modifies actor position, momentum; may trigger collisions
- **Notes:** Likely handles velocity integration, wall bouncing

### MissileTryMove
- **Signature:** `boolean MissileTryMove(objtype*, int, int, int)`
- **Purpose:** Attempt to move projectile; check collision validity
- **Inputs:** Actor, target x/y/z deltas
- **Outputs/Return:** Success boolean
- **Notes:** Collision-aware movement; returns false if blocked

### SelectChaseDir
- **Signature:** `void SelectChaseDir(objtype*)`
- **Purpose:** Pathfinding during active combat pursuit
- **Inputs:** Actor (enemy) object
- **Outputs/Return:** None
- **Side effects:** Updates actor direction and momentum
- **Notes:** Called from CHASE state

### SelectDodgeDir
- **Signature:** `void SelectDodgeDir(objtype*)`
- **Purpose:** Evasive movement away from player/projectiles
- **Inputs:** Actor object
- **Outputs/Return:** None
- **Side effects:** Modifies movement direction
- **Notes:** Called during threatened state

### HeatSeek
- **Signature:** `void HeatSeek(objtype*)`
- **Purpose:** Heat-seeking projectile guidance toward target
- **Inputs:** Projectile actor
- **Outputs/Return:** None
- **Side effects:** Adjusts projectile angle/momentum toward target
- **Notes:** Used by smart missiles; respects angle limits (MAXDELTAYZSEE)

### CheckDoor / NextToDoor
- **Signature:** `boolean CheckDoor(objtype*, doorobj_t*, int, int)`; `boolean NextToDoor(objtype*)`
- **Purpose:** Detect proximity to doors and validate door interaction
- **Inputs:** Actor, door object, coordinates
- **Outputs/Return:** Boolean (door nearby/accessible)
- **Notes:** Collision detection for door interactions

### ActivateEnemy
- **Signature:** `void ActivateEnemy(objtype*)`
- **Purpose:** Transition enemy from idle to active attack state
- **Inputs:** Enemy actor
- **Outputs/Return:** None
- **Side effects:** Sets FL_ATTACKMODE flag, may trigger sounds

### TurnActorIntoSprite
- **Signature:** `void TurnActorIntoSprite(objtype*)`
- **Purpose:** Convert 3D actor to 2D sprite (death/despawn transition)
- **Inputs:** Actor object
- **Outputs/Return:** None
- **Side effects:** Changes rendering mode, may trigger gibs/effects

---

## Macro Utilities (Key Helpers)

| Macro | Purpose |
|-------|---------|
| `M_ISWALL(x)` | Check if collision object is wall type |
| `M_DIST(x1,x2,y1,y2)` | Euclidean distance squared |
| `M_CHOOSETIME(x)` | Frame delay for direction recalculation |
| `M_CHECKDIR(ob,tdir)` | Attempt move in direction; early exit if blocked |
| `M_CHECKTURN(x,ndir)` | Smart turn logic with state transition |
| `STOPACTOR(ob)` | Zero momentum and direction timer |
| `M_CheckDoor(ob)` | Validate and open door if needed |
| `M_CheckBossSounds(ob)` | Boss AI audio/visual effect timing |
| `SET_DEATH_SHAPEOFFSET / RESET_DEATH_SHAPEOFFSET` | Low-violence sprite swapping |

## Control Flow Notes
- **Actor state machine:** Actor moves through discrete states (STAND → CHASE → AIM → FIRE → DIE)
- **Per-frame cycle:** `SelectDir()` → `ParseMomentum()` → `ActorMovement()` → collision checks
- **Boss-specific:** `M_CheckBossSounds` runs during FL_ATTACKMODE to trigger ambient combat audio
- **Low-violence mode:** Conditionally swaps sprite frames via shape offset (gore reduction)

## External Dependencies
- **Actor physics:** References `ParseMomentum()`, `ActorMovement()`, `NewState()` (defined elsewhere)
- **Collision/doors:** `LinkedOpenDoor()`, `doorobjlist[]`, `WallCheck()`
- **Sound:** `SD_PlaySoundRTP()`, BAS actor sound table
- **AI pathing:** `dirangle8[]`, `dirorder[]`, `dirdiff[]` direction lookup tables
- **Random:** `GameRandomNumber()`
- **Rendering:** Shape offset arrays, sprite state definitions
- **Game state:** `gamestate.violence`, `MISCVARS` global timer block
