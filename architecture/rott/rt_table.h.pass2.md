# rott/rt_table.h — Enhanced Analysis

## Architectural Role

This file implements the **global state registry** that underpins ROTT's actor state machine system. It translates symbolic state IDs (used throughout the game) into O(1) runtime lookups of `statetype` definitions, enabling the actor/animation subsystem to execute behavior scripts without conditional branching. The table is the lynchpin connecting the **state definition layer** (states.h, where states are declared) to the **state execution layer** (actor.c, draw.c, where states are animated and executed).

## Key Cross-References

### Incoming (who depends on this file)
- **rt_actor.c / rt_actor.h**: Actor state transitions use state IDs as array indices to fetch the next state's definition (rotation, tictime, think function)
- **rt_draw.c / rt_view.c**: Rendering loop indexes into `statetable[]` by the current actor's state ID to retrieve sprite frame and rotation
- **rt_playr.c**: Player logic indexes states for weapon, movement, and power-up states
- **rt_stat.c**: Animated statics (walls, effects) index their states for animation frames
- **Broadly**: Any `.c` file that manipulates actor state IDs implicitly depends on `statetable` for correctness

### Outgoing (what this file depends on)
- **states.h**: Imports `statetype` type definition and `MAXSTATES` constant
- **All state definition symbols**: References ~1000+ extern `statetype` objects (e.g., `s_lowgrdstand`, `s_explosion1`) defined elsewhere in `.c` files

## Design Patterns & Rationale

**Lookup Table Pattern (Registry):** Instead of switch/case on state IDs or linked lists of states, a pre-allocated, indexed array enables O(1) state lookup. This was essential for 1994 hardware where CPU cycles were precious.

**Conditional Compilation (SHAREWARE flag):** The table omits boss and premium enemy states (~1300 entries) in shareware vs. ~660 in full game, reducing memory footprint. State IDs remain stable across versions—the table just becomes shorter.

**Static Initialization:** The array is initialized at load time; no runtime allocation or linked-list traversal needed. The implicit state ID is the array index—developers must maintain careful order correspondence between the ID constant and the entry position.

**Implicit Index ↔ ID Mapping:** This design assumes state IDs are small integers (0..659 or 0..1299) that directly index the array. There's no validation that a state ID is in bounds—an out-of-range ID causes undefined behavior. This reflects the tight coupling and trust assumptions of 1990s game code.

## Data Flow Through This File

1. **Initialization**: At program start, all ~660–1300 state pointers are resolved and stored in the global `statetable` array.
2. **Runtime Query**: When an actor's state changes (or during rendering/animation), code fetches `statetable[current_state_id]` to obtain the `statetype*`.
3. **State Execution**: The retrieved `statetype` struct contains:
   - `rotate`: Rotation/facing direction
   - `shapenum`: Sprite frame index
   - `tictime`: Animation duration
   - `think`: Function pointer to execute actor logic
   - `condition`: State transition predicate
   - `next`: Next state ID (which becomes the next array index)

## Learning Notes

**Idiomatic 1990s Design:**
- This is a pre-ECS, pre-data-driven approach: behavior is still scattered across think functions, but state data is centralized in a lookup table.
- Modern engines (Unity, Unreal, Godot) use asset pipelines and bytecode interpreters; ROTT uses hard-coded C function pointers and fixed arrays.
- The design assumes states are created at compile time, not loaded at runtime—flexibility comes from tweaking arrays and recompiling, not data-driven configuration.

**Why This Works:**
- Guards, enemies, and effects are defined once at compile time; state machines are stateless templates.
- Indexing by ID is fast and predictable—no memory fragmentation or pointer chasing.
- Symmetry: each distinct actor behavior has a state ID, and that ID directly indexes the table.

**Potential Brittleness:**
- If a state ID is used but not defined or the table is corrupted, the game crashes silently (no bounds checking).
- Adding or reordering states requires careful maintenance of ID constants elsewhere in the codebase.

## Potential Issues

1. **No Bounds Checking**: Indexing into `statetable` with an invalid state ID causes out-of-bounds memory access. This could be a vulnerability if actor state is externally influenced (e.g., network packets) without validation.
2. **Silent Dependencies**: Code that assumes state IDs are sequential or grouped by enemy type (e.g., lowgrd states, then highgrd states) is brittle if the table is reordered.
3. **Conditional Compilation Coupling**: The shareware build may silently access undefined states if code paths aren't properly gated; state IDs are version-specific.
