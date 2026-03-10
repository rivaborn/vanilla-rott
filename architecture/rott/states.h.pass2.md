# rott/states.h — Enhanced Analysis

## Architectural Role

`states.h` is the foundational behavioral framework for Rise of the Triad's entire actor and entity system. It defines the `statetype` structure—a finite-state machine node that holds animation frames, timing, AI logic callbacks, and state transitions. Every game entity (enemy AI, projectile, visual effect, environmental hazard, player) operates as a linked chain of these states, with the global `statetable` providing fast lookup by index. This is the engine's core abstraction for controlling all entity behavior.

## Key Cross-References

### Incoming (who depends on this file)
*Cross-reference data shows function definitions but not state callers specifically. Inferred from codebase structure:*
- **rott/rt_actor.c** (ActorMovement, Collision, actor state machines) – likely calls state think functions and manages state transitions
- **rott/rt_playr.c** (player control, attacks) – uses player states (`s_player`, `s_pgunattack1`, `s_pmissattack1`, etc.)
- **rott/rt_stat.c** (AnimateWalls, static objects) – uses static effect states (`s_explosion*`, `s_fireunit1`, animated traps)
- **rott/rt_game.c** (gameplay logic) – likely spawns entities with initial states
- Various enemy-specific modules – define their state `think()` functions referenced here

### Outgoing (what this file depends on)
- **develop.h** – for `SHAREWARE` flag and debug configuration
- All state `think()` function implementations – defined in parallel `.c` files (e.g., lowgrd thinks, highgrd thinks, etc.)
- Entity structs that reference state pointers (not shown here but assumed in rt_actor.h/rt_types.h)

## Design Patterns & Rationale

**Finite-State Machine (FSM) with Linked States:**  
Each entity doesn't hold a single state ID; instead, it chains states via `next` pointers. This allows multi-frame animations and behavior sequences without separate state stacks. A guard's "chase" sequence might be: `s_highgrdchase1` → `s_highgrdshoot1` → `s_highgrdshoot3` → back to `s_highgrdchase1`.

**Function Pointer Indirection for Behavior:**  
The `think` field in `statetype` is a function pointer, allowing each state to define custom AI/behavior logic. This avoids a giant switch statement per entity type and lets behavior be compiled per-state rather than per-frame.

**State Extern Declarations (not inline definitions):**  
States are declared here but defined elsewhere (likely in corresponding `.c` files), reducing header compilation dependencies and enabling per-file state groupings (e.g., `lowgrd_states.c` defines all low-guard states).

**Conditional Build Segregation:**  
Shareware vs. full version have different state counts (660 vs. 1300), reflecting cut content. This is handled at compile-time, not runtime, keeping binary bloat minimal.

## Data Flow Through This File

1. **Initialization**: Map loader or game startup spawns an entity and assigns it an initial state pointer (e.g., `&s_lowgrdstand`)
2. **Per-Frame Update**: Main game loop iterates all active entities; for each, calls `current_state→think()` with the entity object
3. **Think Logic**: The think function reads entity state, handles AI/behavior, and potentially changes entity properties
4. **State Transition**: Think function sets `entity→state = state→next` (or another state) to advance the FSM
5. **Rendering**: Sprite/animation layer reads `current_state→shapenum` and `rotate` to draw the correct frame

## Learning Notes

**Idiomatic 1990s Game Engine Pattern:**  
This is a classic frame-based FSM, common in DOS/early 3D engines. Modern engines use ECS (Entity-Component-System) or behavior trees, which decouple data from behavior and allow runtime composition. ROTT's linked-list approach is simpler but less flexible (e.g., hard to merge two behavior trees at runtime).

**Naming Conventions Are Self-Documenting:**  
State names like `s_blitzplead1` → `s_blitzdie1` tell a clear story: the boss Blitz entity has a pleading animation before death. This reduces the need for external FSM diagrams.

**`shapenum = -1` as Dynamic Dispatch:**  
Allowing states to specify `shapenum = -1` and read frame data from `entity→temp1` is a clever way to handle variable-length animations without creating separate states per animation length.

**Condition Field is Under-Utilized:**  
The `condition` field (signed char) in `statetype` is declared but the first-pass analysis doesn't explain its role—likely a debugging aid or unused legacy field.

## Potential Issues

1. **State Definition Sprawl**: 600+ extern declarations make this header difficult to navigate and maintain. No grouping by entity type at the declaration level (comments help, but not structure).

2. **Transition Logic Scattered**: State transitions are hard-coded in individual `think()` functions, making it difficult to visualize or modify FSM graphs without reading each think function.

3. **No State Validation**: There's no mechanism to check if a state pointer is valid or detect cycles in the FSM at runtime—undefined behavior if a think function sets `next` to a garbage pointer.

4. **Function Pointer Overhead**: Indirect calls via `think()` every frame add CPU overhead compared to direct function calls; negligible on modern hardware, but notable on 1995 DOS machines under tight frame budgets.

---

*Note: Full cross-reference analysis limited by provided context not including explicit state consumer calls. Inferred from file naming conventions and entity types present in extern declarations.*
