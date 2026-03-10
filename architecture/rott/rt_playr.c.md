# rott/rt_playr.c

## File Purpose
Player and character control system for the ROTT engine. Manages player object lifecycle, input polling from multiple device types, weapon systems, powerup/special modes (god mode, dog mode, etc.), collision with items, and local/network multiplayer player state synchronization.

## Core Responsibilities
- Player object initialization, spawning, revival, and death handling
- Input polling and processing (keyboard, mouse, joystick, VR devices, Cyberman)
- Player movement and physics (momentum, gravity, collision detection)
- Weapon firing, switching, and ammo management
- Powerup system (god mode, dog mode, shrooms, fleet feet, protections)
- Item pickup and bonus application
- Player-environment interaction (doors, switches, platforms)
- Special game modes (tag game, network capture flag)
- Audio feedback for player actions and state changes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `playertype` | struct (linked to objtype) | Per-player state: health, weapons, powerups, animation frame counters, button state, targeting |
| `objtype` | struct | Actor object; players are objtype with obclass=playerobj |
| `specials` | struct | Configuration for powerup durations and respawn times |
| `ROTTCHARS` | array of struct | Character definitions with stats (hitpoints, accuracy, height, sprite offsets) |
| `attack_t` | struct | Weapon attack frame data (attack type, frame number, duration) |
| `williamdidthis` | struct | Weapon/animation sequence definition (frames, timings, attack triggers) |
| `missile_stats` | struct | Projectile weapon parameters (object class, speed, state, offset) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PLAYER[MAXPLAYERS]` | objtype* array | global | Actor pointers for each player |
| `PLAYERSTATE[MAXPLAYERS]` | playertype array | global | Player state data for each player |
| `player` | objtype* | global | Current local player actor (set by consoleplayer) |
| `locplayerstate` | playertype* | global | Current local player state |
| `gamestate` | gametype | global | Global game state (mode, difficulty, counts, battle options) |
| `DEADPLAYER[MAXDEAD]` | statobj_t* array | global | Recycled dead body sprites |
| `BulletHoles[MAXBULLETS]` | statobj_t* array | global | Recycled bullet hole sprites |
| `WEAPONS[MAXWEAPONS]` | williamdidthis array | global | Weapon animation/attack definitions |
| `characters[5]` | ROTTCHARS array | static | Character stat definitions (Taradino, Thi, Doug, Lorelei, Ian) |
| `CurrentSpecialsTimes` | specials | static | Powerup duration timings (60 VBLCOUNTER = 1 second default) |
| `GRAVITY` | int | global | Gravity constant |
| `controlbuf[3]` | int array | global | Combined input from all sources (x, y, turn) |
| `buttonpoll[NUMBUTTONS]` | boolean array | global | Button press state this frame |
| `KX, KY, MX, MY, JX, JY, CX, CY, VX, VY` | int | global | Accumulated input from keyboard, mouse, joystick, cyberman, VR |
| `DoubleClickTimer`, `DoubleClickCount`, `DoubleClickPressed` | arrays | static | Mouse double-click tracking |

## Key Functions / Methods

### T_Player
- **Signature:** `void T_Player(objtype *ob)`
- **Purpose:** Main think function for player actor each frame. Integrates all player systems: movement, weapon state changes, special abilities, collision detection, user input processing.
- **Inputs:** Player actor object
- **Outputs/Return:** None; modifies actor and player state in-place
- **Side effects:** Calls Thrust(), CheckPlayerSpecials(), CheckWeaponChange(), Cmd_Fire(), Cmd_Use(), updates state machine, plays sounds
- **Calls:** Thrust, CheckPlayerSpecials, PlayerMove, NewState, CheckWeaponStates, CheckWeaponChange, Cmd_Use, Cmd_Fire
- **Notes:** Called once per frame for each active player. Handles state machine transitions (s_player, s_pgunattack1, s_pmissattack1, s_tag, etc.). If player is dying, calls T_DeadWait instead.

### PollControls
- **Signature:** `void PollControls(void)`
- **Purpose:** Aggregate input from all input devices and update global control buffers. Called once per frame to collect and normalize user input before physics update.
- **Inputs:** None; reads from global device state (keyboard, mouse, joystick, VR)
- **Outputs/Return:** None; modifies global `buttonpoll[]`, `controlbuf[]`, `KX/KY/MX/MY/JX/JY/CX/CY/VX/VY`
- **Side effects:** I/O to input devices via PollKeyboardButtons, PollMouseButtons, PollJoystickButtons, PollCyberman, PollAssassin, PollVirtualReality
- **Calls:** PollKeyboardButtons, PollMouseButtons, PollJoystickButtons, PollCyberman, PollAssassin, PollVirtualReality, PollKeyboardMove, PollMouseMove, PollJoystickMove, PollMove, UpdateClientControls
- **Notes:** Handles dead player input suppression. Processes pause/exit commands. Builds `buttonbits` for network transmission. Integrates look-up/look-down aim button behavior.

### SpawnPlayerobj
- **Signature:** `void SpawnPlayerobj(int tilex, int tiley, int dir, int playerindex)`
- **Purpose:** Create a new player actor in the world at a starting position. Allocates actor, links player state, initializes all attributes.
- **Inputs:** Tile coordinates (tilex, tiley), direction (0-3), player index (0-numplayers)
- **Outputs/Return:** None; creates global PLAYER[playerindex] and PLAYERSTATE[playerindex]
- **Side effects:** Calls GetNewActor, MakeActive, SetupPlayerobj, NewState; allocates new actor; sets player pointer if local player
- **Calls:** GetNewActor, MakeActive, SetupPlayerobj, NewState
- **Notes:** Initial state is s_player (or s_serialdog if SpawnEluder mode). Respawning players use RevivePlayerobj instead.

### RevivePlayerobj
- **Signature:** `void RevivePlayerobj(int tilex, int tiley, int dir, objtype*ob)`
- **Purpose:** Respawn a dead player actor. Cleans up corpse, plays spawn effects, resets weapons and health.
- **Inputs:** Tile position, direction, player actor object
- **Outputs/Return:** None; modifies player actor state
- **Side effects:** Calls RemoveFromArea, TurnActorIntoSprite, SpawnParticles, RemoveStatic, SetupPlayerobj, ConnectAreas, ResetPlayerstate, InitializeWeapons, NewState, DrawPlayScreen, plays spawn sound
- **Calls:** RemoveFromArea, TurnActorIntoSprite, SpawnParticles, SpawnNewObj (indirectly), SetupPlayerobj, ConnectAreas, ResetPlayerstate, InitializeWeapons, SD_PlaySoundRTP, NewState
- **Notes:** Corpse becomes a sprite if falling into pit; otherwise stored in DEADPLAYER[] for later recycling. Battle respawn has special handling for Tag mode.

### PlayerMove
- **Signature:** `void PlayerMove(objtype *ob)`
- **Purpose:** Update player position based on control input and physics. Applies momentum, gravity, checks for falling, updates light level.
- **Inputs:** Player actor
- **Outputs/Return:** None; modifies actor position, momentum, and state
- **Side effects:** Calls ActorMovement, PlayerTiltHead, UpdateLightLevel, CheckFlying; may update playstate on fall into pit; plays land sound
- **Calls:** ActorMovement, PlayerTiltHead, UpdateLightLevel, CheckFlying, IsWindow, KillActor, NewState, Collision, M_CheckPlayerKilled
- **Notes:** Window collision kills player instantly. Integrates rotation, look angle, and falling checks.

### Thrust
- **Signature:** `void Thrust(objtype *ob)`
- **Purpose:** Main per-frame physics and interaction pass. Calls PlayerMove, applies platforming/tile triggers, checks item pickup, handles environment hazards (pits, heat grates), manages multi-actor links (riding, pillars).
- **Inputs:** Player actor
- **Outputs/Return:** None; modifies actor and surrounding state
- **Side effects:** Calls PlayerMove, checks touchplates and triggers TRIGGER[], calls GetBonus on item pickup, plays sounds, applies damage, modifies gamestate (secretcount, dipballs, etc.), may call NewState, Collision, BATTLE functions
- **Calls:** PlayerMove, GetBonus, Collision, DamageThing, M_CheckPlayerKilled, NewState, BATTLE_PlayerKilledPlayer, BATTLE_CheckGameStatus, Move_Player_From_Exit_To_Start, UpdateKills, SD_PlaySoundRTP
- **Notes:** Handles capture-the-flag pickup, pit falling, heat grate damage, column/pillar riding. Checks exit conditions (EXITTILE, SECRETEXITTILE). Extensive multi-condition logic for complex interactions.

### Cmd_Fire
- **Signature:** `void Cmd_Fire(objtype*ob)`
- **Purpose:** Handle attack/fire button press. Initiate weapon attack animation and state change based on current weapon type.
- **Inputs:** Player actor
- **Outputs/Return:** None; changes player state and weapon frame counters
- **Side effects:** Calls NewState, may call SD_PlaySoundRTP
- **Calls:** NewState
- **Notes:** Checks for weapon change in progress (W_CHANGE). Special handling for NETCAPTURED state (knife attack). Different states for bullet weapons (s_pgunattack1), missile weapons (s_pmissattack1), dog/bat weapons. Updates attackcount and weaponframe.

### Cmd_Use
- **Signature:** `void Cmd_Use(objtype*ob)`
- **Purpose:** Handle use/interact button. Determine direction player faces, find target object (door, switch, pillar, pushwall), execute appropriate action.
- **Inputs:** Player actor
- **Outputs/Return:** None; may modify door state, trigger switches, push walls, operate elevators, spawn items
- **Side effects:** Plays sounds, updates tilemap[], calls AddMessage, NewState, OperateDoor, OperateElevatorDoor, OperatePushWall, SD_PlaySoundRTP, game state updates
- **Calls:** Cmd_Use (for dog mode), NewState, OperateDoor, OperateElevatorDoor, OperatePushWall, OperatePushWall, AddMessage, SD_PlaySoundRTP, SD_Play
- **Notes:** Cardinal direction logic based on player angle. Extensive branching for different object types (doors with locks/keys, elevator switches, push walls, pillars, switches). Handles Tag game state switch.

### PlayerMissileAttack
- **Signature:** `void PlayerMissileAttack(objtype*ob)`
- **Purpose:** Fire a missile weapon (bazooka, heatseeker, godhand, etc.). Spawn projectile(s), apply auto-targeting if applicable, play fire sound, handle ammo depletion.
- **Inputs:** Player actor
- **Outputs/Return:** None; creates new missile actor(s)
- **Side effects:** Calls SpawnMissile, MissileAutoTarget, MissileTryMove, plays fire sound, modifies player ammo count, updates HUD
- **Calls:** SpawnMissile, MissileAutoTarget, MissileTryMove, SD_PlaySoundRTP, DrawBarAmmo
- **Notes:** Auto-targeting for godhand and kes weapons. Drunk missile spawns 4 projectiles. Singular weapons (bazooka, heatseeker) stored in PLAYER0MISSILE global. Missile camera support (missilecam).

### GunAttack
- **Signature:** `void GunAttack(objtype *ob)`
- **Purpose:** Fire a bullet weapon (pistol, dual pistol, MP40). Perform raycast, apply damage, optionally auto-target enemy.
- **Inputs:** Player actor
- **Outputs/Return:** None; damage applied to hit targets
- **Side effects:** Calls RayShoot, AutoTargetHorizon, plays fire sound, may update targeting
- **Calls:** AutoTargetHorizon, RayShoot, SD_PlaySoundRTP
- **Notes:** Damage varies by weapon. Accuracy adjusted by character stat + difficulty. Auto-targeting for horizon/vertical aim.

### GetBonus
- **Signature:** `void GetBonus(objtype*ob,statobj_t *check)`
- **Purpose:** Apply item/powerup effects when player picks it up. Extensive switch on item type: weapons, health, keys, modes (god/dog/shrooms), protections.
- **Inputs:** Player actor, static object (item)
- **Outputs/Return:** None; modifies player state, may remove item sprite
- **Side effects:** Plays sound, updates health/ammo/keys, changes player flags, calls GiveWeapon, GiveMissileWeapon, GivePowerup, GiveProtection, SpawnNewObj (flash effect), NewState, AddMessage, game state updates, may change item to empty variant
- **Calls:** GiveWeapon, GiveMissileWeapon, GivePowerup, GiveProtection, GivePoints, GiveKey, GiveLives, HealPlayer, UpdateTriads, NewState, SpawnNewObj, RemoveStatic, AddMessage, SD_PlaySoundRTP, GetBonusTimeForItem, SetPlayerHorizon, ResetWeapons, SaveWeapons
- **Notes:** Huge function with ~50 item types. Random powerup generator. Battle mode special handling (weapon persistence, ammo rules). Respawning items can be disabled.

### CheckPlayerSpecials
- **Signature:** `void CheckPlayerSpecials(objtype *ob)`
- **Purpose:** Update all player special states and timers each frame: powerup expirations, protections, special sounds, recording, height offsets.
- **Inputs:** Player actor
- **Outputs/Return:** None; modifies player state and actor flags
- **Side effects:** Calls CheckRemoteRecording, CheckTemp2Codes, CheckSpecialSounds, CheckProtectionsAndPowerups; plays sounds; updates flags; may reset weapons
- **Calls:** CheckRemoteRecording, CheckTemp2Codes, CheckSpecialSounds, CheckProtectionsAndPowerups
- **Notes:** Core integration point for all special player state logic. Called from T_Player and T_Attack.

### CheckProtectionsAndPowerups
- **Signature:** `void CheckProtectionsAndPowerups(objtype *ob, playertype *pstate)`
- **Purpose:** Handle expiration of powerups and protections. Revert special modes, clear flags, play expiration sounds when timers reach zero.
- **Inputs:** Player actor and state
- **Outputs/Return:** None; modifies actor flags and player state
- **Side effects:** Plays sounds, calls ResetWeapons, NewState, GM_UpdateBonus, updates HUD
- **Calls:** ResetWeapons, NewState, GM_UpdateBonus, SD_PlaySoundRTP
- **Notes:** Separate logic for powerups (FL_SHROOMS, FL_ELASTO, FL_FLEET, FL_GODMODE, FL_DOGMODE) and protections (FL_BPV, FL_AV, FL_GASMASK). God/dog mode exit has gender-specific sounds.

### CheckWeaponChange
- **Signature:** `void CheckWeaponChange(objtype *ob)`
- **Purpose:** Handle weapon switching via hotkeys or swap button. Check if new weapon is available, initiate weapon-down animation.
- **Inputs:** Player actor
- **Outputs/Return:** None; modifies player weapon state
- **Side effects:** Calls StartWeaponChange macro, plays sound, updates HUD
- **Calls:** GiveWeapon, StartWeaponChange (macro), SD_Play, DropWeapon, PlayNoWaySound
- **Notes:** Multiple input methods: dedicated keys (bt_pistol, bt_dualpistol, bt_mp40, bt_missileweapon), swap button (toggle bullet/missile), drop weapon (battle mode). Weapon availability checks via HASBULLETWEAPON[].

### CheckWeaponStates
- **Signature:** `void CheckWeaponStates(objtype*ob)`
- **Purpose:** Update weapon animation height each frame during raise/lower animations. Decrement weaponuptics/weapondowntics timers.
- **Inputs:** Player actor
- **Outputs/Return:** None; modifies weapon height and timer
- **Side effects:** Updates actor's weapon display; may update HUD
- **Calls:** None directly (modifies state)
- **Notes:** Called from T_Player and T_Attack. Special handling for kes weapon (extra height adjustment). Battle Tag mode has different logic.

### PlayerTiltHead
- **Signature:** `void PlayerTiltHead(objtype *ob)`
- **Purpose:** Update player vertical look angle (yzangle). Handle look-up/look-down input, auto-target horizon tracking, falling tilt, shroom sway, gun target following.
- **Inputs:** Player actor
- **Outputs/Return:** None; modifies actor yzangle
- **Side effects:** Calls SetPlayerHorizon, SetNormalHorizon, updates horizon state based on target or input
- **Calls:** SetPlayerHorizon, SetNormalHorizon
- **Notes:** Complex priority: auto-target > look buttons > falling tilt > shrooms sway > normal horizon. Snap-back behavior when returning to normal. Limits to YZANGLELIMIT.

### CheckFlying
- **Signature:** `void CheckFlying(objtype*ob,playertype *pstate)`
- **Purpose:** Handle Mercury Mode (fleet feet) flight input. Apply upward/downward momentum on look-up/look-down buttons.
- **Inputs:** Player actor and state
- **Outputs/Return:** None; modifies actor momentum
- **Side effects:** Updates momentumz
- **Calls:** None directly
- **Notes:** Inverts standard look mechanics: look-up/down control altitude. FLYINGZMOM = 350000 (large momentum).

### T_Attack
- **Signature:** `void T_Attack(objtype *ob)`
- **Purpose:** Main attack state think function. Manage weapon animation frames, fire attacks, handle attack-to-idle transitions, weapon swapping during attack.
- **Inputs:** Player actor
- **Outputs/Return:** None; modifies actor state, weapon frames, ammo
- **Side effects:** Calls Thrust, CheckPlayerSpecials, CheckWeaponStates, Cmd_Use, GunAttack, PlayerMissileAttack, DogAttack, DogBlast, BatAttack, plays sounds, updates HUD, NewState
- **Calls:** Thrust, CheckPlayerSpecials, NewState, CheckWeaponStates, Cmd_Use, GunAttack, PlayerMissileAttack, DogAttack, DogBlast, BatAttack, SD_PlaySoundRTP, StartVRFeedback, StopVRFeedback, SetIllumination, DrawBarAmmo, Error
- **Notes:** Executes attack data from WEAPONS[].attackinfo[]. Switch on attack type (at_pulltrigger, at_missileweapon, reset, reset2). Handles ammo checks, special weapon logic (dog blast charge, bat blast charge, kes weapon). Player can cancel mid-attack if no ammo.

### DogAttack
- **Signature:** `void DogAttack(objtype*ob)`
- **Purpose:** Execute dog mode bite attack. Find nearby targets in range, apply melee damage with knockback.
- **Inputs:** Player actor
- **Outputs/Return:** None; damage applied to target
- **Side effects:** Plays dog bite sound, calls DamageThing, Collision, BATTLE_PlayerKilledPlayer, returns immediately after hitting first valid target
- **Calls:** SD_PlaySoundRTP, DamageThing, Collision, BATTLE_PlayerKilledPlayer
- **Notes:** ~0xc000 radius, range checks on dx/dy/dz. Returns on first hit. Checks actor type, SHOOTABLE flag, and DYING flag.

### BatAttack
- **Signature:** `void BatAttack(objtype*ob)`
- **Purpose:** Execute Excalibat attack. Find and hit/deflect all nearby objects (actors, grenades, missiles) and destructible sprites/walls.
- **Inputs:** Player actor
- **Outputs/Return:** None; damage/collision applied
- **Side effects:** Plays swing sound, applies momentum to targets, can deflect grenades, plays hit sounds, updates masked walls, calls DamageThing, Collision, NewState
- **Calls:** SD_PlaySoundRTP, AngleBetween, FixedMul, FixedSqrtHP, GetMomenta, ParseMomentum, NewState, DamageThing, Collision, UpdateMaskedWall, Error (implicit)
- **Notes:** Extensive logic for grenade deflection and momenta calculation. Checks blitz guard special state. ~0x10000 range search.

## Control Flow Notes
This file integrates into the main game loop as follows:

**Per-Frame Sequence:**
1. `PollControls()` — Aggregate all input sources, build buttonpoll[] and controlbuf[]
2. `T_Player()` — Main think function; processes state machine, calls Thrust() for physics/collision, calls CheckPlayerSpecials() for timers/effects
3. `CheckWeaponChange()` — Weapon selection logic (called from T_Player)
4. `Cmd_Fire()` / `Cmd_Use()` — Handle specific actions triggered by input
5. `T_Attack()` — If in attack state, execute attack animation and fire logic
6. `PlayerMove()` / `Thrust()` — Physics, collision, item pickup
7. `PlayerTiltHead()` — Update look angle
8. `CheckPlayerSpecials()` → `CheckProtectionsAndPowerups()` → Update HUD/sounds on mode changes

**Powerup/Effect Timeline:**
- Entry: Item pickup in `Thrust()` → `GetBonus()` → Sets flag and `pstate->poweruptime`
- Active: `CheckProtectionsAndPowerups()` decrements timer each frame
- Exit: Timer reaches 0, clears flag, plays expiration sound, may reset weapons (god/dog mode)

**Weapon State Machine:**
- Idle state: s_player (can press fire)
- Fire pressed: `Cmd_Fire()` → `NewState(ob, &s_pgunattack1)` or `&s_pmissattack1`
- Attack state: `T_Attack()` advances attackframe on timer expiry
- On attack event (at_pulltrigger/at_missileweapon): Call `GunAttack()` or `PlayerMissileAttack()`
- Reload/idle: attackframe reaches numattacks, state returns to s_player

**Death Sequence:**
- Player takes lethal damage: flags |= FL_DYING
- Next `T_Player()`: Calls `PlayerMove()` → `CheckPlayerSpecials()` → early return
- Death state: s_remoteguts12 / s_remotemove (animated death)
- Final: `T_DeadWait()` → playstate = ex_died (single-player) or respawn (multiplayer)

## External Dependencies
- Notable includes / imports:
  - `rt_def.h` — Global constants, enum types (playerobj, weapons, states, buttons)
  - `rt_sound.h` — Sound constants and functions (SD_PlaySoundRTP, SD_PlayPitchedSound)
  - `rt_actor.h` — Actor functions (presumably DamageThing, Collision, KillActor)
  - `rt_main.h` — Main loop integration (presumably playstate, gamestate, ticcount)
  - `rt_game.h` — Game mode and battle functions (BATTLEMODE, BATTLE_PlayerKilledPlayer, BATTLE_CheckGameStatus)
  - `rt_view.h` — Camera/viewport (SetIllumination, UpdateLightLevel)
  - `rt_door.h` — Door operations (OperateDoor, OperateElevatorDoor)
  - `rt_menu.h` — Menu/UI (AddMessage, GM_DrawBonus, DrawBarAmmo)
  - `rt_draw.h` — Drawing/HUD (DrawPlayScreen, DrawTriads, DrawBarAmmo)
  - `rt_ted.h` — Map/tile operations (MAPSPOT, tilemap, DiskAt)
  - `rt_swift.h` — VR/Cyberman (SWIFT_Get3DStatus)
  - `z_zone.h` — Memory management (GetNewActor, MakeActive)
  - `states.h` — State definitions (statetype, s_player, s_pgunattack1, etc.)
  - `sprites.h` — Sprite data (presumably BAS[], stats[])
- External symbols used but not defined here:
  - `PLAYER`, `PLAYERSTATE`, `gamestate`, `DEADPLAYER`, `BulletHoles` — global arrays/structs
  - `GetNewActor()`, `MakeActive()`, `RemoveStatic()`, `SpawnInertActor()`, `SpawnInertStatic()`, `NewState()` — actor creation/state
  - `ActorMovement()`, `ActorTryMove()`, `MissileTryMove()` — physics
  - `RayShoot()`, `CheckLine()` — weapon raycast
  - `DamageThing()`, `Collision()`, `KillActor()` — damage/death
  - `OperateDoor()`, `OperateElevatorDoor()`, `OperatePushWall()` — environment
  - `FindDistance()`, `atan2_appx()`, `ParseMomentum()`, `AngleBetween()` — math
  - Sound functions: `SD_Play()`, `SD_PlaySoundRTP()`, `SD_SetSoundPitch()`, `SD_StartRecordingSound()`, etc.
  - Input: `IN_UpdateKeyboard()`, `IN_GetMouseButtons()`, `IN_JoyButtons()`, `INL_GetJoyDelta()`, `Keystate[]`
  - Multiplayer: `UpdateClientControls()`, `AddRespawnCommand()`, `AddPauseStateCommand()`, `AddExitCommand()`
  - HUD: `AddMessage()`, `GM_DrawBonus()`, `DrawBarAmmo()`, `DrawTriads()`, `DrawPlayScreen()`
  - Map: `ConnectAreas()`, `RemoveFromArea()`, `MakeLastInArea()`, `TurnActorIntoSprite()`, `PlatformHeight()`, `IsPlatform()`, `IsWindow()`
