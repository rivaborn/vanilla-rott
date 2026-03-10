# rott/rt_state.c

## File Purpose
Defines all finite state machine states for game entities (enemies, hazards, effects, player). Each state specifies animation frame (sprite), duration, behavior function, and transition to next state.

## Core Responsibilities
- Declare static state structures for 50+ entity types
- Define state chains for animation sequences (standing, walking, attacking, dying)
- Link state behavior functions (T_Chase, T_Path, A_Shoot, etc.)
- Organize state transitions for actor state machines
- Support both shareware and full-game entity sets via conditional compilation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| statetype | struct | State machine node: sprite, tics, behavior function, next state |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| s_lowgrdstand, s_lowgrdpath1, ... | statetype | static/extern | State definitions for Low Guard enemy |
| s_explosion1 through s_explosion20 | statetype | static/extern | Explosion animation sequences |
| s_player, s_pgunattack1, ... | statetype | static/extern | Player states (idle, shooting, moving) |
| s_darianstand, s_darianchase1, ... | statetype | static/extern | Boss Darian state chains |
| s_darkmonkstand, s_darkmonkchase1, ... | statetype | static/extern | Dark Monk boss states |
| s_NMEstand, s_NMEchase, ... | statetype | static/extern | Orobot/NME boss states |

## Key Functions / Methods
None—this is a data-only file. All behavior is defined through function pointers in `statetype.think` fields (e.g., `T_Chase`, `A_Shoot`, `T_Projectile`), which are declared as `extern` and implemented elsewhere.

## Control Flow Notes
- **Actor State Machine**: Each actor object (`objtype`) holds a `state` pointer pointing to the current `statetype`.
- **Per-Frame Update**: `DoActor()` (in rt_actor.c) calls the `state->think` function, which updates position/animation and may transition `state` via `NewState()`.
- **State Chains**: Death sequences chain together (e.g., `s_lowgrddie1` → `s_lowgrddie2` → `s_lowgrddead`). Walk animations cycle (e.g., `s_lowgrdpath1` → `s_lowgrdpath2` → `s_lowgrdpath3` → `s_lowgrdpath4` → `s_lowgrdpath1`).
- **Initialization**: States are bundled into a `statetable[]` array (from states.h) for index lookups during game load/serialization.

## External Dependencies
- **sprites.h**: `SPR_LOWGRD_W41`, `SPR_EXPLOSION1`, etc.—sprite/shape IDs
- **states.h**: `statetype` struct definition, `MAXSTATES` constant, state declarations
- **rt_def.h**: `TILESHIFT`, `TILEGLOBAL`, tile/map definitions
- **rt_actor.h**: Actor behavior functions (`T_Chase`, `A_Shoot`, `T_Projectile`, etc.)
- **Behavior Functions** (defined elsewhere): `T_Stand`, `T_Path`, `T_Chase`, `A_Shoot`, `T_Collide`, `T_Explode`, `T_Projectile`, `ActorMovement`, `T_Roll`, `T_BossDied`, etc.

**Notes:**
- File is 4000+ lines; all content is state declarations with no functional code.
- Conditional `#if (SHAREWARE == 0)` sections exclude boss/special entity states in shareware version.
- States reference function pointers like `T_Player`, `T_NME_Explode`, `T_DarkmonkChase`—these are defined in rt_actor.c or other behavior modules.
- `condition` field (signed char) uses flags like `SF_CLOSE`, `SF_CRUSH`, `SF_SOUND` to tag special state properties.
