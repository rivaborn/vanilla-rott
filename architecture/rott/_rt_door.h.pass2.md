# rott/_rt_door.h — Enhanced Analysis

## Architectural Role

This private header is the **data definition core** for the door/touch plate subsystem. While `rt_door.h` likely exports public functions (`ConnectAreas`, `CheckTile`), `_rt_door.h` defines the **persistent state structure and animation timing constants** that these functions manipulate. It bridges level-design data (tile flags like `FL_TACT`, `FL_TSTAT`) with runtime mechanics by embedding state machines into `saved_touch_type` instances. The tic-based timing framework (OPENTICS, AMW_TICCOUNT) synchronizes door/wall animations with the game loop.

## Key Cross-References

### Incoming (who depends on this file)
- **rott/rt_door.c**: Main implementation module that uses `saved_touch_type` to manage touch plate lifecycles and calls `ConnectAreas()`, `CheckTile()`
- **rott/_rt_acto.h / rott/rt_actor.c**: Actor collision code (`CheckDoor`, `NextToDoor`) reads door state to detect player-door interactions
- **rott/rt_door.c** likely stores arrays of `saved_touch_type` for active touch plates in the current map

### Outgoing (what this file depends on)
- **None**: This is pure data—no external includes, no function calls. It defines only constants and types.

## Design Patterns & Rationale

| Pattern | Implementation | Why |
|---------|---|---|
| **State Machine** | `triggered` → `complete` → `done` flags | Allows door animation to span multiple frames; a single boolean isn't enough |
| **Tic-Based Animation** | `tictime` (duration), `ticcount` (elapsed) | 1990s game engines used discrete "tics" (1/70th sec) rather than delta-time |
| **Action Indirection** | `actionindex`, `swapactionindex` | Actions are behavior codes (likely indices into a behavior table), allowing level designers to reuse logic |
| **Bitflag Integration** | `FL_TACT` (0x4000), `FL_TSTAT` (0x8000) | Encode door properties directly in tile data to avoid separate lookup tables |
| **Dual-State Toggle** | Two action indices for open/close pairs | Supports "press plate → open door; press again → close" without extra state |

Why structured this way: **Space-efficient for 1990s DOS/console memory budgets**. Bitflags in tile data save lookup table space; tic-based timing avoids floating-point math.

## Data Flow Through This File

```
Touch Plate Trigger Cycle:
  Level Data (tile with FL_TACT flag)
    ↓
  [Actor steps on plate]
    ↓
  saved_touch_type instantiated with:
    - actionindex = what behavior to execute
    - triggered = true (frame N)
    - ticcount = 0, tictime = duration (OPENTICS or custom)
    ↓
  [Game loop: CheckTile() or door update function]
    ↓
  Increment ticcount each frame
    ↓
  When ticcount >= tictime:
    - complete = true (animation finished)
    - Trigger swapactionindex if needed (toggle logic)
  ↓
  done = true (clean up this entry)
```

**Door Animation**: OPENTICS = 165 frames → ~2.3 seconds at 70 FPS (typical early 90s standard).  
**Push Wall Animation**: 9 frames × 3 tics = 27 tics visible per frame.

## Learning Notes

**1990s Game Engine Idioms:**
- **No delta-time**: Fixed 70 FPS tick used for all timing; hard-coded constants like OPENTICS assume this clock
- **Action tables**: `actionindex` is probably an offset into a master behavior table (not shown), allowing designers to compose behavior without code
- **Level-data encoding**: Bitflags (`FL_TACT`, `FL_TSTAT`) compress metadata into 16-bit tile word; no separate property files needed
- **Tic budgeting**: NUMTOUCHPLATEACTIONS (8) likely limits concurrent touch plate activations—suggests limited state memory

**How modern engines differ:**
- Use delta-time or frame-independent timing
- Entity-component systems replace hard-coded state machines
- Scripting/blueprint systems replace action index tables

**Connections to game engine concepts:**
- Similar to **Doom's sector tag system**: level data encodes which behaviors activate (Doom used sector IDs + line actions; ROTT uses tile flags + action indices)
- Early predecessor to **trigger volumes** in modern 3D engines

## Potential Issues

1. **Unbounded ticcount**: No max check visible. If tictime is never reached (corrupted data?), ticcount could overflow. Likely handled in rt_door.c.
2. **whichobj type ambiguity**: Stored as `int` but header gives no clue what object types are valid. Requires reading rt_door.c or game_objects.h to understand.
3. **Hard-coded animation speeds**: OPENTICS and AMW_TICCOUNT baked into header; tuning door speed or wall animation requires recompilation.
4. **Single swapaction limitation**: Only two action indices—blocks more complex state machines (e.g., 3-way toggle). Would need array expansion.
5. **No initialization macro**: No `#define INIT_TOUCH_PLATE { ... }` helper, so initialization code might be error-prone.

---

**Note:** Architecture context did not load fully; these inferences are based on cross-reference clues (CheckDoor, ConnectAreas, rt_door.c) and typical 1990s game architecture patterns. Confirmation would require reading rt_door.c and rt_actor.c.
