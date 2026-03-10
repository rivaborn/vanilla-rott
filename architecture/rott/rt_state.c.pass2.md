# rott/rt_state.c — Enhanced Analysis

## Architectural Role

This file is the **state registry** for the entire entity subsystem—a static lookup table of all finite-state-machine nodes used by actors (enemies, bosses, effects, player). Every entity in the game follows a state chain defined here. The file serves as the binding layer between the *data* (sprite animations) and the *behavior* (think functions like `T_Chase`, `A_Shoot`) that drive gameplay logic. It is essential to the actor-based entity system that powers combat, NPC AI, and visual effects.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_actor.c** (`DoActor()`, `NewState()`, collision handlers): Reads actor `state` pointers and calls their `think` functions each frame; transitions between states via pointers defined here.
- **rt_game.c** / game initialization: Likely references state chains when spawning enemies or transitioning to new levels.
- **rt_playr.c** (player control): Uses player states (`s_player`, `s_pgunattack1`, etc.) to animate and control the player entity.
- **Serialization/Load code**: State pointers are serialized into save games; state indices may be looked up from a global state table.
- **Boss/special entity spawners**: Code that creates Darian, Dark Monk, NME, Kratchy, etc. references their initial states (`s_darianstand`, `s_darkmonkstand`).

### Outgoing (what this file depends on)
- **sprites.h**: Every state references sprite constants (`SPR_LOWGRD_W41`, `SPR_EXPLOSION1`, etc.). This file is a heavy consumer of sprite IDs.
- **states.h**: Defines the `statetype` struct and may export a global `statetable[]` array for state lookups.
- **rt_actor.h**: All `think` function pointers (e.g., `T_Chase`, `A_Shoot`, `ActorMovement`) are declared `extern` here and implemented in rt_actor.c or related modules.
- **rt_def.h**: Likely provides constants and type definitions used implicitly.

## Design Patterns & Rationale

**Finite State Machine via Linked Structures:**  
Each `statetype` is a node in a directed, typically cyclic graph. States chain via the last field (e.g., `s_lowgrdpath1` → `s_lowgrdpath2` → ... → `s_lowgrdpath4` → `s_lowgrdpath1`). This creates **animation loops** and **behavior sequences** without explicit loops in code.

**Function Pointer Dispatch:**  
Instead of a large switch statement, behavior is decoupled via `think` function pointers. Different enemy types share state *structure* but differ in behavior (e.g., `s_lowgrdchase1` calls `T_Chase`, but `s_blitzchase1` also calls `T_Chase`—likely with different enemy attributes for variation).

**No Functional Logic:**  
The file is *purely declarative*. All control flow (animations, collision, state transitions) happens in the `think` functions elsewhere. This is a **data-driven design** common in 90s engines.

**Entity-Type Organization:**  
States are grouped by enemy type (Low Guard, High Guard, Strike Guard, Blitz, Enforcer, Robo Guard, Bosses, Effects). Within each type, subgroups by activity (stand, path, chase, shoot, die, crushed). This reflects a hierarchical entity taxonomy.

**Death Sequences:**  
Each enemy has a multi-step death chain (`s_lowgrddie1` → `s_lowgrddie2` → ... → `s_lowgrddead`). Crushed variants branch off early, suggesting environment-specific death handling.

## Data Flow Through This File

1. **Initialization**: Game startup loads the state table (implicitly via `states.h` or explicit array initialization).
2. **Entity Spawning**: When an enemy spawns, it is assigned an initial state pointer (e.g., `obj->state = &s_lowgrdstand`).
3. **Per-Frame Update**: `DoActor()` (in rt_actor.c) invokes `obj->state->think(obj)`, which:
   - Animates the sprite (incrementing frame counters).
   - Moves the actor (if applicable).
   - Checks for collisions and state transitions.
   - Calls `NewState(obj, next_state)` to advance the state chain.
4. **Damage/Death**: On hit, an actor transitions to a pain or death state.
5. **Serialization**: State pointers may be stored as indices during save/load.

## Learning Notes

**Idiomatic to this engine & era:**
- **No explicit AI trees or behavior systems**: Just linked state nodes. Modern engines use behavior trees or hierarchical state machines; this is flatter and more memory-efficient.
- **Sprite-centric animation**: Each state is *one sprite ID*, not a sprite *sequence*. Animation progress is managed by `tics` (frame duration) and automatic chaining.
- **Tight coupling of animation and behavior**: You cannot separate "how it looks" from "what it does"—the state node couples sprite, tics, and think function.
- **Enemy variation via attributes, not states**: Two guards might use the same states but differ in speed/health (stored in `objtype`), not in state definitions.

**Design philosophy:**
- **Declarative over imperative**: States describe a graph; logic is embedded in think functions.
- **Reuse via pointers**: Walk animations are shared across multiple enemy types (e.g., `SPR_LOWGRD_W41` used by both low and high guards).
- **Memory-efficient**: State nodes are small (~40 bytes each); many entities can reference the same state.

**Connections to broader concepts:**
- **Finite State Machine**: Classic FSM pattern, though implemented via static data rather than state class hierarchy.
- **Entity Component System (ECS) precursor**: The `objtype` + `statetype` pair is an early form of data-driven entity design, though not as flexible as modern ECS.

## Potential Issues

1. **No bounds checking on state chains**: If a `think` function corrupts the `next` pointer, the game could crash or loop infinitely. Modern engines validate FSM transitions.
2. **Hard-coded state pointers in code**: Other modules directly reference states (e.g., `NewState(obj, &s_lowgrddie1)`). Refactoring is error-prone; a state registry with string keys would be safer.
3. **Sprite assumption**: Each state assumes a unique sprite ID. If two sequential frames share a sprite, the code must artificially split them into separate states with 1-tic duration—fragile and verbose.
4. **Limited editor support**: States are hand-coded; no visual state editor or toolchain to validate transitions or catch orphaned states.
5. **Shareware branching**: Conditional compilation (`#if SHAREWARE == 0`) bakes in two different state sets. Modern engines would load dynamically.

---

**Summary**: rt_state.c is the **declaration layer** of a classic actor-based game engine. It exemplifies early 90s data-driven design: simple, memory-efficient, but inflexible compared to modern engines. Its heavy reliance on external `think` functions and tight sprite-animation coupling reflects the performance constraints and toolchain limitations of the era.
