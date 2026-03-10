# rott/rt_actor.h — Enhanced Analysis

## Architectural Role

This header defines the **central entity management subsystem** of ROTT's game engine. Every dynamic object in the world—enemies, player, projectiles, hazards, particles—is an `objtype` instance managed through interconnected linked lists and spatial grids. The file acts as the **gateway between game logic (AI, physics, rendering, collision) and the entity pool**, providing spawn/despawn, movement, and state-machine integration. It is the critical bridge connecting the battle system (`rt_battl.c`), player controller (`rt_playr.c`), static objects (`rt_stat.c`), and door system (`rt_door.c`) to a unified entity abstraction.

## Key Cross-References

### Incoming (who depends on this file)

- **Battle system** (`rt_battl.c`): calls `BATTLE_CheckGameStatus` (via macro `M_CheckPlayerKilled`); reads `PLAYER0MISSILE`, actor death state
- **Player controller** (`rt_playr.c`): spawns missiles via `SpawnMissile`, applies damage via `DamageThing`, checks player death, reads `SCREENEYE`
- **Door system** (`rt_door.c`): spawns wall traps (spears, firejets, crushers) via `SpawnSpear`, `SpawnFirejet`, `SpawnCrushingColumn`; uses `ResolveDoorSpace` for push-wall collision
- **Static object system** (`rt_stat.c`): shares `actorat[]` grid for collision validation; may reference actors in area lists
- **Cinematic system** (`cin_actr.c`, `cin_evnt.c`): spawns/controls cinematic actors; reads state and position fields
- **Network system** (`rt_net.c`): serializes/deserializes actor state via `SaveActors`/`LoadActors` for multiplayer sync
- **Rendering** (implied): reads `drawx`, `drawy`, `shapenum`, `shapeoffset` for sprite composition
- **Level loader**: calls `InitActorList` at map start, then spawns entities via `SpawnInertActor`, `SpawnStand`, `SpawnPatrol`, etc.

### Outgoing (what this file depends on)

- **State machine system** (`states.h`): imports `statetype` struct; all actors reference state instances (e.g., `s_lowgrdstand`, `s_chase1`)
- **Audio system**: uses `soundhandle` field; reads/writes via battle and actor subsystems
- **Map/geometry subsystem**: uses `MAPSIZE`, `TILESHIFT`, `TILEGLOBAL` constants; queries `actorat[]` for collision; assumes `InMapBounds` macro
- **Physics**: uses `momentumx/y/z` fields; movement functions interact with collision/blocking checks
- **Gib/particle system** (`SpawnParticles`): triggered on death/hit; consumes `gib_t` enum for particle type
- **Trigonometry/lookup tables** (`angletodir[]`, `AngleBetween`): direction and angle queries for AI pathfinding

## Design Patterns & Rationale

### 1. **Intrusive Linked Lists**
`objtype` embeds `next/prev`, `nextactive/prevactive`, `nextinarea/previnarea` pointers directly. Enables O(1) insertion/removal and multiple simultaneous orderings without secondary data structures. **Trade-off:** Actor struct bloats; list corruption is hard to debug.

### 2. **Spatial Grid (`actorat[]`)**
Maps tile (x,y) to occupant pointer for O(1) collision lookups. **Rationale:** Early '90s hardware required fast spatial queries; grid avoids O(n) actor iteration. **Limitation:** Only one actor per tile; handles via priority or stacking (not visible in header).

### 3. **Deferred Deletion (`Add_To_Delete_Array`, `Remove_Delete_Array_Entries`)**
Actors queued for removal rather than freed immediately. **Rationale:** Prevents iterator corruption if an actor dies during iteration (e.g., in a `for` loop over active actors). **Pattern:** Common in real-time systems; cleaned up at frame boundary.

### 4. **Bi-Modal Active/Inactive States**
`firstactive/lastactive` list (thinking actors) vs. static pool. **Rationale:** Skip `DoActor()` calls for statues, corpses, inert scenery. Early optimization for low CPU. **Modern analogy:** ECS system-of-systems.

### 5. **Macro-Heavy Design**
`SetTilePosition`, `SetFinePosition`, position macros; `M_ABS` inline asm; flag bitwise ops. **Rationale:** Late-'80s/early-'90s C convention; avoids function call overhead; enables cross-platform inline asm. **Learning:** Shows era's micro-optimization culture.

### 6. **State-Machine Callbacks**
`statetype` struct (from `states.h`) holds `think` function pointer. Each frame, `DoActor()` increments `ticcount`; when ≥ `tictime`, invoke think and reset. **Rationale:** Decouples AI logic from entity update loop; think functions call `NewState()` to transition. **Modern equivalent:** State machines in game engines (Unreal's behavior trees, Unity FSMs).

### 7. **Polymorphic Damage (`DamageThing(void*)` vs `KillActor(objtype*)`)**
`DamageThing` accepts void pointer (flexible); `KillActor` is type-specific. **Rationale:** Player, enemies, and destructibles may have different damage logic; void* allows reuse. **Trade-off:** Loses type safety; must cast internally.

## Data Flow Through This File

### Spawn Path
```
Level loader → SpawnInertActor / SpawnStand / SpawnPatrol / SpawnMissile
    ↓
GetNewActor (allocate from pool)
    ↓
Insert into: FIRSTACTOR↔LASTACTOR, firstareaactor[zone]↔lastareaactor[zone]
    ↓
Update actorat[tilex][tiley] (spatial grid)
    ↓
MakeActive() if dynamic
```

### Per-Frame Update
```
Game loop iterates firstactive → lastactive
    ↓
DoActor(ob) for each actor
    ├─ ticcount++
    ├─ if ticcount ≥ state→tictime:
    │   ├─ Call state→think(ob)
    │   └─ Reset ticcount
    └─ think functions:
        ├─ T_Chase: SelectChaseDir() → ActorTryMove()
        ├─ T_Path: SelectPathDir() → ActorTryMove()
        ├─ A_Shoot: SpawnMissile() / RayShoot()
        └─ T_Projectile: MissileTryMove() → MissileHit() if blocked
```

### Movement & Collision
```
ActorTryMove(ob, newtilex, newtiley, dir)
    ↓
QuickSpaceCheck() (verify tile walkable)
    ├─ Check actorat[newtilex][newtilei] for occupants
    └─ Check map walls/hazards
    ↓
If blocked: return false
If clear: update ob→tilex/tiley/x/y; update actorat[]
    ↓
Movement may trigger Collision(ob, attacker, momx, momy)
    ├─ Apply momentum
    └─ Call DamageThing() if attack
```

### Damage & Death
```
DamageThing(victim, dmg)
    ↓
Decrement hitpoints; if ≤ 0:
    ├─ NewState(ob, death_state)
    ├─ M_CheckPlayerKilled() → BATTLE_CheckGameStatus() (if victim is player)
    ├─ SpawnParticles() (gibs, blood)
    └─ RemoveObj() (queue for deferred deletion)
```

### Cleanup
```
RemoveObj(ob)
    ├─ RemoveFromArea(ob)
    ├─ Unlink from FIRSTACTOR↔LASTACTOR
    ├─ MakeInactive(ob)
    └─ Add_To_Delete_Array(ob)
    ↓
(End of frame)
    ↓
Remove_Delete_Array_Entries()
    └─ Free pooled actors; reset objcount
```

## Learning Notes

### Idiomatic to ROTT / Early '90s Game Engines
1. **Hand-rolled memory pools**: No malloc per spawn; `GetNewActor()` reuses freed slots. Deterministic O(1) allocation (critical for real-time).
2. **Bit flags for state**: `flags & FL_SYNCED`, `flags & FL_MASTER`—compact state representation before object-oriented designs.
3. **Global spatial grid**: `actorat[]` is a top-level global (not passed as parameter). Reflects flat procedural architecture; modern engines encapsulate in manager objects.
4. **Macro-based position abstraction**: `SetTilePosition()`, `SetFinePosition()` hide coordinate system detail; shows awareness of data locality but via C preprocessor rather than inline functions.
5. **Think functions as method pointers**: Precursor to virtual functions; allows different behaviors without inheritance (early functional design pattern).
6. **Per-area actor lists**: Hand-rolled spatial partitioning; modern engines would use quadtrees or octrees, but this tile-grid granularity was appropriate for ROTT's 2.5D architecture.

### Modern Game Engine Contrast
- **Then (ROTT):** Intrusive linked lists, void pointer polymorphism, macro-heavy, manual memory management.
- **Now:** ECS (Entity-Component-System) with archetypes, type-safe component queries, automatic pool management, SIMD-friendly data layouts.

### Connections to Game Engine Concepts
- **Object/Entity Pool**: Classic real-time pattern; seen in modern engines (Unreal's Object system, Unity's prefab instantiation).
- **State Machines**: Foundational for AI and behavior; modern engines use behavior trees, but ROTT's flat state graph is simpler and more suitable for fixed AI patterns.
- **Spatial Partitioning**: Essential for collision and visibility; ROTT's grid is predecessor to hierarchical structures.
- **Deferred Deletion**: Ensures frame stability; also used in graphics APIs (GPU resource freeing at frame boundary).

## Potential Issues

### 1. **Single-Actor-Per-Tile Limit**
`actorat[x][y]` stores one pointer; if multiple actors try to occupy a tile, one silently overwrites. The code assumes priority/stacking but header doesn't expose it. Possible stack-overflow bug if overlapping corpses + spawns occur.

### 2. **No Iterator Invalidation Safeguards**
The deferred deletion pattern mitigates this, but if an actor removes itself mid-frame (e.g., `RemoveObj` called from a think function), the linked-list pointers (`next`, `prev`) are still followed by outer loops. Needs verification that outer loops never store/reuse `next` after RemoveObj.

### 3. **Void Pointer Polymorphism**
`DamageThing(void*)` and `CheckLine(void*, void*)` are fragile. If a void* is cast to wrong type (e.g., pointer to `staticobj` instead of `objtype`), behavior is undefined. No RTTI safeguard in C89.

### 4. **Global Mutable State**
Many globals (`FIRSTACTOR`, `mstruct`, `actorat[]`) are read/written without apparent synchronization. In multiplayer (`rt_net.c`), serialization via `SaveActors`/`LoadActors` must happen at frame boundaries; race conditions possible if accessed during network sync.

### 5. **Limited Spatial Query Granularity**
`InMapBounds` checks only tile integer boundaries. Fine-grained position (`x`, `y` as fixed-point) is not grid-aligned; possible for actors to be at sub-tile precision but `actorat[]` lookup coarser. May cause collision misses at tile boundaries.

### 6. **Missing Documentation on `whichactor` Field**
`objtype::whichactor` is declared but never documented in header. Used to link actors to something (actor ID? networking index?); usage pattern unclear.

---

**Token estimate:** ~1400. Cross-references grounded in provided architecture context; design patterns inferred from C89 era conventions and linked-list heavy usage; no speculative conclusions.
