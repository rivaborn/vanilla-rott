# rott/rt_game.c

## File Purpose
Core game UI and state management module that handles HUD rendering (status bar, health, ammo, score), game progression mechanics (death/respawn, level completion with bonuses), save/load systems, and high score tracking. Implements both single-player campaign and multiplayer battle mode UI.

## Core Responsibilities
- **HUD Rendering**: Status bar, health bar, ammo counter, score, lives, keys, time, and powerup indicators
- **Score System**: Points tracking, lives management, triads (bonus item counter), high score management
- **Game Progression**: Level completion with bonus calculations, death sequences with visual effects, game over handling
- **Battle Mode UI**: Kill tallies, player rankings, death counts for multiplayer modes
- **Save/Load**: Full game state serialization/deserialization with checksums
- **Visual Effects**: Screen shake, damage-based border color shifts, death camera rotation sequences
- **Bonus/Stats Display**: End-of-level scoring screens, multiplayer end-game statistics

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `STR` | struct | String with cached length (for rapid redraw) |
| `HighScore` | struct | Single high score entry (name, score, completion level) |
| `gamestorage_t` | struct (in header) | Complete saved game state snapshot |
| `pic_t` | typedef (external) | Planar VGA picture format |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PlayerSnds[5]` | int array | global | Sound IDs for player death sounds by character |
| `Scores[MaxScores]` | HighScore array | global | High score table |
| `SHAKETICS` | int | global | Screen shake timer (0xFFFF = inactive) |
| `damagecount` | int | global | Damage state for border flash effect |
| `SaveTime` | int | global | Timestamp of last save action |
| `lifeptnums[10]`, `lifenums[10]`, etc. | pic_t* arrays | static | Cached digit graphics for display |
| `health[6]`, `ammo[26]` | pic_t* arrays | static | Cached health bar and ammo indicator sprites |
| `men[5]`, `fragpic[5]` | pic_t* arrays | static | Player portrait and frag box graphics |
| `poweruptime`, `protectiontime` | int | static | Active powerup timers |
| `powerupheight`, `protectionheight` | int | static | Powerup display height (for partial rendering) |
| `EndLevelStuff` | boolean | static | Flag: true during level completion sequence |
| `playeruniformcolor` | int | static | Current player's team color index |
| `oldplayerhealth`, `oldpercenthealth` | int | static | Previous frame health (for dirty rect optimization) |

## Key Functions / Methods

### SetupPlayScreen
- **Signature**: `void SetupPlayScreen(void)`
- **Purpose**: Initialize and cache all HUD graphics elements from WAD lumps at level start
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Allocates and caches ~40 pic_t graphics (digits, icons, bars); modifies static pic_t* arrays
- **Calls**: `W_CacheLumpName()`, `W_CacheLumpNum()`, `CacheLumpGroup()` (WAD system)
- **Notes**: Called once per level; handles both single-player and battle mode graphics setup; different ammo graphics loaded for shareware vs full version

### DrawPlayScreen
- **Signature**: `void DrawPlayScreen(boolean bufferofsonly)`
- **Purpose**: Render entire HUD each frame (status bar, health, ammo, score, keys, powerups)
- **Inputs**: `bufferofsonly` – if true, draw only to primary buffer; if false, draw to triple-buffered pages
- **Outputs/Return**: None
- **Side effects**: Renders to video memory; modifies display state; calls other draw functions
- **Calls**: `DrawTime()`, `DrawBarHealth()`, `DrawBarAmmo()`, `DrawKeys()`, `DrawLives()`, `DrawScore()`, `DrawKills()`, `GameMemToScreen()`, etc.
- **Notes**: Conditional rendering based on `SHOW_TOP_STATUS_BAR()`, `SHOW_BOTTOM_STATUS_BAR()`, `BATTLEMODE` macros; powerup visual height animated during display

### Died
- **Signature**: `void Died(void)`
- **Purpose**: Handle complete player death sequence—optional cinematic death camera, facing killer, respawn or game over
- **Inputs**: None (uses global `player`, `gamestate`, `locplayerstate`)
- **Outputs/Return**: None
- **Side effects**: Resets player state; decrements lives; may trigger game over; updates UI; plays sounds; spawns dummy body for camera
- **Calls**: `ZoomDeathOkay()`, `SpawnPlayerobj()`, `UpdateGameObjects()`, `ThreeDRefresh()`, `RotateBuffer()`, `VL_FadeOut()`, `InitializeWeapons()`, `ResetPlayerstate()`, `RotateBuffer()` (with various fade effects)
- **Notes**: Complex multi-path logic: cinematic zoom-out if conditions met, else rotation to face attacker, else instant fade; "dopefish" mode enables chaotic dummy body physics; slowrate parameter affects animation timing

### LevelCompleted
- **Signature**: `void LevelCompleted(exit_t playstate)`
- **Purpose**: Execute end-of-level sequence: display stats, calculate and award bonuses, animate score updates
- **Inputs**: `playstate` – exit reason (ex_completed, ex_secretdone, ex_bossdied, etc.)
- **Outputs/Return**: None
- **Side effects**: Updates `gamestate.score` with bonus calculations; renders multiple screens sequentially; plays sounds; may trigger final "BONUS BONUS" screen
- **Calls**: `DrawEOLHeader()`, `DrawEndBonus()`, `CheckHolidays()`, `GetNextMap()`, `AddMessage()`, `SD_Play()`, `MU_StartSong()` (music), `VL_FadeIn()`, `VW_UpdateScreen()`
- **Notes**: Computes 11 different bonus categories (adrenaline, bull in china shop, curiosity, ground zero, etc.) with ratio checks; displays running score updates; blocks on user input; can loop back for special "BONUS BONUS" if all bonuses earned

### SaveTheGame
- **Signature**: `boolean SaveTheGame(int num, gamestorage_t *game)`
- **Purpose**: Serialize complete game state to disk with CRC integrity check
- **Inputs**: `num` – save slot (0–15); `game` – pre-allocated game state struct
- **Outputs/Return**: true on success, false if insufficient disk space
- **Side effects**: Creates/overwrites save file; allocates temporary buffers; validates disk space; calculates and appends CRC
- **Calls**: `_dos_getdiskfree()`, `SafeOpenWrite()`, `SaveDoors()`, `SaveElevators()`, `SavePushWalls()`, `SaveMaskedWalls()`, `SaveSwitches()`, `SaveStatics()`, `SaveActors()`, `SaveTouchPlates()`, `MU_SaveMusic()`, `CalculateSaveGameCheckSum()`
- **Notes**: Structured as sequential tag+data pairs (ROTT header, DOOR, ELEVATOR, etc.); validates ~120KB minimum disk space; saves gamestate, playerstate (per player), song, misc state (ticcount, shaketics, powerup heights)

### LoadTheGame
- **Signature**: `boolean LoadTheGame(int num, gamestorage_t *game)`
- **Purpose**: Deserialize saved game from disk, restore full game state, validate integrity
- **Inputs**: `num` – save slot (0–15); `game` – pre-allocated struct
- **Outputs/Return**: true on success, false on version/CRC mismatch or missing tags
- **Side effects**: Frees current level resources; rebuilds entire level from saved state; restores player position, weapons, ammo; reinitializes rendering and music
- **Calls**: `LoadFile()`, `DoCheckSum()`, `Z_FreeTags()`, `SetupGameLevel()`, `LoadDoors()`, `LoadElevators()`, etc. (symmetric to Save), `SetViewSize()`, `ConnectAreas()`, `PreCache()`, `InitializeMessages()`, `CalcTics()`
- **Notes**: Validates CRC; allows "corrupted" save recovery with user prompt; rebuilds game world from components; recalculates lighting; must reinitialize player camera and subsystems

### DrawKills
- **Signature**: `void DrawKills(boolean bufferofsonly)`
- **Purpose**: Render battle mode kill/score display: current player box, "it" player, kill-count ranking table
- **Inputs**: `bufferofsonly` – triple-buffer control
- **Outputs/Return**: None
- **Side effects**: Renders frag boxes, kill counts, player names, team indicators
- **Calls**: `GetShortCodeName()`, `DrawPPic()`, `DrawGameString()`, `StatusDrawColoredPic()`, `DrawNumber()`, `W_CacheLumpName()`
- **Notes**: Conditional display based on `SHOW_TOP_STATUS_BAR()` and `SHOW_KILLS()`; renders up to `MAXKILLBOXES` (10) teams; handles team vs deathmatch mode; shows negative points with special graphics

### BattleLevelCompleted
- **Signature**: `void BattleLevelCompleted(int localplayer)`
- **Purpose**: Interactive end-game stats screen for battle modes—cycle between final score, kills, deaths via arrow keys
- **Inputs**: `localplayer` – player index
- **Outputs/Return**: None
- **Side effects**: Loops until user presses Esc; can switch displayed player with Shift+arrows
- **Calls**: `VL_DrawPostPic()`, `ShowEndScore()`, `ShowKills()`, `ShowDeaths()`, `ReadAnyControl()`, `MN_PlayMenuSnd()`
- **Notes**: Three-page display; shows WhoKilledWho matrix; ranks players by score or kills; highlights local player row; allows inspection of any player's stats

### DrawMPPic
- **Signature**: `void DrawMPPic(int xpos, int ypos, int width, int height, int heightmod, byte *src, boolean bufferofsonly)`
- **Purpose**: Render a planar VGA sprite with per-pixel masking (0xFF = transparent) and optional vertical height clipping
- **Inputs**: Coordinates, dimensions, heightmod (offset for multi-plane source), source data pointer, bufferofsonly flag
- **Outputs/Return**: None
- **Side effects**: Writes directly to video memory (bufferofs or triple-buffer pages)
- **Calls**: `VGAMAPMASK()` (VGA register setup), direct memory writes
- **Notes**: Implements 4-plane VGA rendering with plane masking; processes one plane per loop iteration; heightmod used for partial-height rendering (health bar animation); transparent pixel (255) skipped in output

### ScreenShake
- **Signature**: `void ScreenShake(void)`
- **Purpose**: Apply screen jitter effect when taking damage
- **Inputs**: None (reads global `SHAKETICS`, `tics`)
- **Outputs/Return**: None
- **Side effects**: Modifies `displayofs` (screen memory offset) to create jitter
- **Calls**: `RandomNumber()` (seed-based random)
- **Notes**: Inactive when `SHAKETICS == 0xFFFF`; decrements timer each frame; selects random offset ±1 pixel or ±3*linewidth

### CheckHighScore
- **Signature**: `void CheckHighScore(long score, word other, boolean INMENU)`
- **Purpose**: Evaluate if score qualifies for high score table, insert if so, prompt for name entry
- **Inputs**: `score` – final score; `other` – completion level; `INMENU` – whether called from menu vs gameplay
- **Outputs/Return**: None
- **Side effects**: May insert into `Scores[]` array; triggers menu UI; plays sounds; saves name input
- **Calls**: `MenuFadeIn()`, `SetupMenuBuf()`, `DisplayInfo()`, `DrawHighScores()`, `US_LineInput()` (name input), `RefreshMenuBuf()`
- **Notes**: Shifts down lower scores on insertion; differentiates display based on entry point (menu vs end-of-game)

## Control Flow Notes
**Init/Frame/Shutdown**:
- `SetupPlayScreen()` called once at level start to cache graphics
- `DrawPlayScreen()` called every frame during gameplay to render HUD
- Game death handled by `Died()` → respawn or game over
- Level completion handled by `LevelCompleted()` → bonus display → next level
- Save/load system bridges play sessions via `SaveTheGame()` / `LoadTheGame()`

**Battle Mode**: `DrawKills()` renders per-frame, `BattleLevelCompleted()` executes post-game

**Screen Effects**: `ScreenShake()` and `DoBorderShifts()` apply per-frame visual feedback

## External Dependencies
- **WAD/Resource System**: `W_CacheLumpName()`, `W_CacheLumpNum()`, `W_GetNumForName()` (w_wad.h, lumpy.h)
- **Video/Graphics**: `VL_MemToScreen()`, `GM_MemToScreen()`, `VWB_DrawPic()`, `VL_FadeOut()`, `VL_FadeIn()`, `VGAMAPMASK()` (rt_vid.h, modexlib.h)
- **Audio**: `SD_Play()`, `SD_PlaySound()`, `SD_PlaySoundRTP()`, `SD_SoundActive()`, `MU_StartSong()`, `MU_SaveMusic()`, `MU_LoadMusic()` (rt_sound.h, isr.h)
- **Physics/World**: `UpdateGameObjects()`, `ThreeDRefresh()`, `CalcTics()`, `GetMapCRC()` (rt_main.h, engine.h)
- **Input**: `IN_CheckAck()`, `IN_ClearKeysDown()`, `IN_UpdateKeyboard()`, `ReadAnyControl()` (rt_in.h, rt_menu.h)
- **UI/Menu**: `DrawMenuBufPropString()`, `VW_DrawPropString()`, `SetMenuTitle()`, `DisplayInfo()` (rt_menu.h)
- **Actors/Objects**: `SpawnPlayerobj()`, `Collision()`, `SpawnStatic()`, `LoadPlayer()` (rt_actor.h, rt_playr.h)
- **Utility**: `SafeMalloc()`, `SafeRead()`, `SafeWrite()`, `StringsNotEqual()` (watcom.h)
- **Battle System**: `BATTLE_SetOptions()`, `BATTLE_Init()`, `BATTLE_Team[]`, `BATTLE_Points[]`, `WhoKilledWho[]` (rt_battl.h)

**Defined Elsewhere**:
- Global: `player`, `locplayerstate`, `PLAYERSTATE[]`, `gamestate`, `numplayers`, `displayofs`, `bufferofs`, `screenofs`, `ticcount`, `timelimitenabled`, `timelimit`, `demoplayback`, `tedlevel`, `screenfaded`, `viewsize`, `consoleplayer`
- Macros: `SHOW_TOP_STATUS_BAR()`, `SHOW_BOTTOM_STATUS_BAR()`, `SHOW_KILLS()`, `BATTLEMODE`, `ARMED()`, `M_LINKSTATE()`, `LASTSTAT`, `FIRSTACTOR`
