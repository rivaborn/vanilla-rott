# rott/rt_playr.c — Enhanced Analysis

## Architectural Role

This file is the **engine's main player controller subsystem**—a central hub that integrates input aggregation, physics, collision, item pickup, weapon firing, and game state into a unified per-frame update loop. It bridges between human/network input and the rest of the engine (actor system, sound, rendering, multiplayer) by managing player state as both a struct (`playertype`) and an actor object (`objtype` with `obclass=playerobj`). The file is responsible for translating raw device input into gameplay, coordinating state machines (idle → attack → recover), and synchronizing player state across local and networked instances.

## Key Cross-References

### Incoming (who calls this file's functions)

- **Main game loop** → calls `PollControls()` and `T_Player()` once per frame (called via actor think function dispatch, likely from `rt_main.c` or `rt_actor.c`)
- **Actor spawning** → `rt_actor.c` calls `SpawnPlayerobj()` and `RevivePlayerobj()` during map initialization and respawn
- **Item system** → `rt_stat.c` triggers `GetBonus()` when player walks over static objects (doors, weapons, health pickups)
- **Weapon/attack dispatch** → `rt_actor.c` state machine calls `T_Player()` and `T_Attack()` via `statetype` function pointers
- **Damage/death** → `rt_actor.c` calls `Collision()` → may update player health or set `FL_DYING` flag
- **Battle system** → `rt_battl.c` calls `BATTLE_PlayerKilledPlayer()` from within `DogAttack()` and `Thrust()` for multiplayer scoring

### Outgoing (what this file depends on)

- **Input devices** → calls `PollKeyboardButtons()`, `PollMouseButtons()`, `PollJoystickButtons()`, `PollCyberman()`, `PollAssassin()`, `PollVirtualReality()` (input subsystem)
- **Physics/collision** → calls `ActorMovement()`, `ActorTryMove()`, `Collision()`, `CheckLine()` (actor subsystem in `rt_actor.c`)
- **Weapon firing** → calls `RayShoot()` (raycast for bullets, in `rt_actor.c` or `rt_draw.c`), `SpawnMissile()` (for projectiles)
- **Sound** → calls `SD_PlaySoundRTP()`, `SD_Play()`, `SD_SetSoundPitch()`, `SD_StartRecordingSound()` (audio subsystem)
- **Game state** → reads/writes `gamestate` (global); calls `BATTLE_PlayerKilledPlayer()`, `BATTLE_CheckGameStatus()` from `rt_battl.c`
- **Map/environment** → calls `ConnectAreas()`, `RemoveFromArea()`, `MakeLastInArea()`, `OperateDoor()`, `OperateElevatorDoor()`, `OperatePushWall()` from `rt_door.c`
- **Door system** → reads `MAPSPOT()`, `tilemap[]`, `EXITTILE`, `SECRETEXITTILE` to detect exits and interact with doors
- **Rendering** → calls `DrawPlayScreen()`, `UpdateLightLevel()`, `DrawBarAmmo()`, `DrawTriads()` from `rt_draw.c` and `rt_menu.c`
- **Multiplayer sync** → calls `UpdateClientControls()`, `AddRespawnCommand()`, `AddPauseStateCommand()` from `rt_net.c`
- **Memory management** → calls `GetNewActor()`, `MakeActive()`, `RemoveStatic()`, `SpawnNewObj()`, `SpawnParticles()`, `SpawnInertActor()` from zone/actor managers
- **State machine** → calls `NewState()` to transition between `statetype` definitions (s_player, s_pgunattack1, s_pmissattack1, s_tag, etc.)

## Design Patterns & Rationale

**State Machine with Function Pointers:** Each player is an `objtype` with a `statetype` (e.g., `s_player`, `s_pgunattack1`). The statetype stores a function pointer (`think`) that is called each frame. This decouples player logic from the main loop and makes it trivial to add new player states (e.g., special animations, status effects) without touching core game code.

**Input Normalization as Pipeline:** `PollControls()` aggregates 6+ input sources (keyboard, mouse, joystick, Cyberman arcade cabinet controller, spaceball, VR) into unified x/y/turn vectors stored in globals (KX, KY, MX, MY, JX, JY, etc.). This abstraction layer allows supporting exotic 1990s hardware without sprawling per-device logic throughout the codebase. The normalized output is combined into `controlbuf[3]` and `buttonbits` for network transmission.

**Character as Data (not hardcoded):** The `characters[5]` array stores per-character stats (hitpoints, accuracy, height, sprite offset). This allows the game to support 5 playable characters with different abilities (Taradino is slower but tougher; Thi is faster but weaker) via data lookups rather than branching. New characters could be added by extending this array and sprite/state definitions.

**Powerup System as Timer-based Finite Buffs:** Rather than event-driven expiration, powerup timers (`pstate->poweruptime`, `pstate->protectiontime`) tick down each frame in `CheckProtectionsAndPowerups()`. When a timer reaches zero, the corresponding flag (e.g., `FL_GODMODE`) is cleared and an expiration sound plays. This is simple, deterministic, and predictable for networking (frame-exact).

**Weapon Animation as Data-Driven Structs:** The `williamdidthis WEAPONS[MAXWEAPONS]` array stores animation metadata: frame count, sprite info, and a sequence of `attack_t` commands (e.g., `at_pulltrigger`, `at_missileweapon`, `reset`). This allows `T_Attack()` to be a generic interpreter that handles all weapon types uniformly, and new weapons can be added as data (sprite number, frame timing) rather than new code branches.

**Local vs. Remote Player Separation:** `player` and `locplayerstate` point to the local human-controlled player, while `PLAYER[]` and `PLAYERSTATE[]` arrays hold all players (local and remote). This simplifies multiplayer code: network input is deserialized directly into `PLAYERSTATE[remote_idx]`, and the local player's input is built from `PollControls()`. Each player runs the same `T_Player()` think function, making behavior deterministic across the network.

## Data Flow Through This File

```
User Input (Hardware)
  ↓
PollControls() — aggregates keyboard, mouse, joystick, special devices
  ↓ (outputs: KX/KY/MX/MY, buttonpoll[], controlbuf[])
  ↓
T_Player() — main per-frame think function
  ├─ PlayerMove() — update position based on controlbuf[]
  │   ├─ ActorMovement() — physics and collision checks
  │   └─ UpdateLightLevel() — ambient lighting
  ├─ Thrust() — interactions with environment
  │   ├─ Check tile triggers (TRIGGER[], touchplate)
  │   ├─ Item pickup → GetBonus() → set flags (FL_GODMODE, FL_DOGMODE), ammo, health
  │   ├─ Door/exit detection → OperateDoor(), KillActor() (for window collision)
  │   └─ Pit/hazard damage → DamageThing()
  ├─ CheckPlayerSpecials() → CheckProtectionsAndPowerups() — tick timers, clear expired buffs
  ├─ CheckWeaponChange() — hotkey weapon selection
  └─ Cmd_Fire() / Cmd_Use() — action buttons
      ├─ Cmd_Fire() → NewState(&s_pgunattack1 or &s_pmissattack1)
      └─ Cmd_Use() → OperateDoor(), OperatePushWall(), OperateElevatorDoor()
  ↓
T_Attack() — weapon animation think function (called when in attack state)
  ├─ Advance attackframe timer
  ├─ On attack trigger (at_pulltrigger / at_missileweapon):
  │   ├─ GunAttack() → RayShoot() → hit detection, damage
  │   └─ PlayerMissileAttack() → SpawnMissile() → projectile creation
  └─ On timer expiry → NewState(&s_player) — return to idle

Powerup State Management (parallel with above):
  ├─ GetBonus() — sets pstate->poweruptime and FL_* flags
  ├─ CheckProtectionsAndPowerups() — each frame, decrement timers
  └─ On expiry → clear flag, play sound, possibly reset weapons
```

## Learning Notes

**Per-Frame Tick Architecture:** This engine (mid-1990s) uses a simple **per-frame update loop**: collect input → run physics → check collisions → pickup items → advance animation → draw. No event queues or deferred operations. This is typical of real-time action games of the era and keeps code straightforward and frame-deterministic (important for networking).

**Fixed-Point Math Everywhere:** Angles, momentum, and positions use fixed-point (`angle << ANGLEBITS`, momentum as large integers like `FLYINGZMOM = 350000`). This avoids floating-point overhead on 1990s CPUs and ensures determinism across network play. Modern engines use floats; this reflects hardware constraints of the time.

**Input as Vectors, not Events:** Rather than "key pressed" events, the system builds cumulative x/y/turn vectors from all input sources. This is intuitive for action games (player holds strafe key → sustained rightward motion) and easy to network (transmit final vector, not per-key press).

**Character Progression via Data:** The 5 playable characters have *different stats* (Taradino: 100 HP / 25 skill; Doug: 150 HP / 20 skill). This is implemented as lookups (`characters[pstate->player].hitpoints`), not branching. Teaches the value of separating gameplay data from behavior code.

**Weapon Framework Shared with AI:** The `WEAPONS[]` array and `T_Attack()` think function are **not specific to the player**. Enemy AI uses the same weapon system (see cross-references to AI actors calling `A_Shoot`, `A_MissileWeapon`). This is a best practice: unified attack/weapon handling ensures consistency.

**Multiplayer Determinism via Frame-Locked Input:** Player input is processed once per frame in a fixed order (keyboard, mouse, joystick, etc.). Network packets contain `buttonbits` and `controlbuf[]` snapshots. Remote players' input is deserialized into the same format, so all players execute identical behavior given identical input—crucial for deterministic multiplayer.

**Contrasts with Modern Engines:** A modern engine would use **ECS** (player as entity with components: TransformComponent, HealthComponent, WeaponComponent, PowerupComponent). This monolithic playertype struct is pre-ECS. Modern engines also use **event systems** (InputEvent, DamageEvent) rather than polling and flag checks each frame.

## Potential Issues

**Global State Coupling:** The heavy reliance on globals (`PLAYER[]`, `PLAYERSTATE[]`, `gamestate`, `DEADPLAYER[]`, `BulletHoles[]`) makes unit testing difficult and the subsystem hard to refactor. Adding a new game mode requires modifying multiple global variables scattered across this file.

**Monolithic GetBonus() Function:** This ~50-line switch statement handles all 40+ item types. Adding a new item (e.g., new weapon, new powerup) requires modifying this already-large function and risks breaking existing branches. A cleaner design would be **item-as-object with virtual pickup methods** (though unfeasible in 1990s C without function pointers per item type).

**Input Accumulation Race Condition Risk:** If `PollControls()` is called multiple times per frame (unlikely but not prevented), `KX/KY/MX/MY` will accumulate incorrectly. The function should either be guard-checked (`if (already_polled_this_frame) return;`) or called exactly once per tick. Not visible in this file alone; depends on main loop discipline.

**Fixed-Point Overflow in Mercury Mode:** Flight momentum (`FLYINGZMOM = 350000`) added repeatedly to a 32-bit `momentumz` over many frames could theoretically overflow. This is mitigated by limits in `CheckFlying()` or physics code, but not visible here.

**Dead Code due to Conditional Compilation:** Multiple `#if (SHAREWARE == 0)` blocks suggest features removed for the shareware build. The full-game weapon array includes `wp_split`, `wp_kes`, `wp_bat` conditionally. Porting this code to a modern build system without these #ifdefs would require careful separation of features.

**Hardcoded Angle Thresholds:** Values like `GODYZANGLE = -(9*FINEANGLES/360)` (god mode head tilt) and `SHROOMYZANGLE = (15*FINEANGLES/360)` (shroom sway) are magic numbers. No configuration or easy way to tweak visual feedback for game feel.
