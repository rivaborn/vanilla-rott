# rott/rt_game.h

## File Purpose
Public interface for the game state and UI rendering system. Declares functions for managing gameplay flow, HUD drawing, player damage/health, scoring, save/load mechanics, and game progression callbacks (level completion, death, high scores).

## Core Responsibilities
- HUD rendering (screen, kills, score, lives, keys, time, health/ammo bars, bonus indicators)
- Player state modifications (damage, healing, weapon/item distribution, life management)
- Game state persistence (save/load game data, high score tracking, saved message retrieval)
- Game progression callbacks (level completion, death, screen shake effects)
- Bonus/powerup system management (update and display)
- Pause screen and UI state transitions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `gamestorage_t` | struct | Encapsulates game save state: message, episode, area, version, rendered picture, map CRC, and alternate resource info |
| `HighScore` | struct | Single high score entry: player name, score, completion flag, episode number |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PlayerSnds[5]` | int array | global | Sound handles for player-related sound effects |
| `SHAKETICS` | int | global | Tic counter for screen shake effect duration |
| `damagecount` | int | global | Current accumulated damage counter for flash/indicator logic |
| `Scores[MaxScores]` | HighScore array | global | High score table (7 entries) |
| `SaveTime` | int | global | Timestamp or tic count of last game save |

## Key Functions / Methods

### SetupPlayScreen / DrawPlayScreen
- Signature: `void SetupPlayScreen(void)`, `void DrawPlayScreen(boolean bufferofsonly)`
- Purpose: Initialize play screen layout and render the full HUD each frame
- Inputs: `bufferofsonly` – flag for buffer-only rendering (no screen update)
- Outputs/Return: None
- Side effects: Modifies video buffer, may update global screen state
- Calls: Calls other Draw* functions; depends on global game state
- Notes: Frame-per-frame rendering; `bufferofsonly` parameter suggests double-buffering

### TakeDamage / HealPlayer
- Signature: `void TakeDamage(int points, objtype *attacker)`, `void HealPlayer(int points, objtype *ob)`
- Purpose: Apply damage to or heal the player; update health state
- Inputs: `points` – health delta; `attacker`/`ob` – entity responsible (for killcam or stat tracking)
- Outputs/Return: None
- Side effects: Modifies player health, may trigger screen effects or game over
- Calls: Likely updates player state via external references
- Notes: Asymmetric parameters (attacker vs. ob); healing may be capped

### GiveWeapon / GiveMissileWeapon / GiveKey
- Signature: `void GiveWeapon(objtype *ob, int weapon)`, `void GiveMissileWeapon(objtype *ob, int which)`, `void GiveKey(int key)`
- Purpose: Award weapons, missile weapons, or keys to player
- Inputs: `ob` – player object; `weapon`/`which` – weapon/item ID; `key` – key index
- Outputs/Return: None
- Side effects: Updates player inventory; may trigger UI feedback
- Calls: Updates player state via objtype or global player state
- Notes: Weapon and missile weapon are separate systems; GiveKey takes only key ID

### SaveTheGame / LoadTheGame
- Signature: `boolean SaveTheGame(int num, gamestorage_t *game)`, `boolean LoadTheGame(int num, gamestorage_t *game)`
- Purpose: Persist or restore complete game state to/from save slot
- Inputs: `num` – save slot index; `game` – pointer to save structure (populated on load)
- Outputs/Return: `boolean` – success/failure
- Side effects: File I/O; modifies all player and world state on load
- Calls: Serializes/deserializes actor and player state
- Notes: Bidirectional; caller allocates `gamestorage_t`; `mapcrc` validates level integrity

### CheckHighScore / DrawHighScores
- Signature: `void CheckHighScore(long score, word other, boolean INMENU)`, `void DrawHighScores(void)`
- Purpose: Test if score qualifies for high score table and render leaderboard
- Inputs: `score` – player's final score; `other` – context flag (level/battle mode); `INMENU` – display context
- Outputs/Return: None
- Side effects: May insert into `Scores[]` and trigger save; updates display buffer
- Calls: Likely updates persistent high score state
- Notes: `other` parameter semantics unclear; INMENU controls rendering context

### LevelCompleted / BattleLevelCompleted / Died
- Signature: `void LevelCompleted(exit_t playstate)`, `void BattleLevelCompleted(int localplayer)`, `void Died(void)`
- Purpose: Handle level progression, battle mode completion, and player death
- Inputs: `playstate` – exit code from level; `localplayer` – player index in multiplayer
- Outputs/Return: None
- Side effects: Triggers screen transitions, score updates, game over checks
- Calls: Likely updates global game state machine
- Notes: Separate paths for standard vs. battle mode progression

### UpdateScore / UpdateLives / UpdateTriads
- Signature: `void UpdateScore(int num)`, `void UpdateLives(int num)`, `void UpdateTriads(objtype *ob, int num)`
- Purpose: Increment score, lives, or triad counters and refresh display
- Inputs: `num` – amount to add; `ob` – player object (for triads)
- Outputs/Return: None
- Side effects: Updates global counters and HUD
- Calls: Updates display state
- Notes: `UpdateTriads` takes object reference; others are global

### DrawColoredMPPic / StatusDrawColoredPic
- Signature: `void DrawColoredMPPic(int xpos, int ypos, int width, int height, int heightmod, byte *src, boolean bufferofsonly, int color)`, `void StatusDrawColoredPic(unsigned x, unsigned y, pic_t *nums, boolean bufferofsonly, int color)`
- Purpose: Render color-mapped pictures to HUD (multiplayer graphics, status elements)
- Inputs: Position, dimensions, heightmod, source buffer, color index
- Outputs/Return: None
- Side effects: Modifies video buffer
- Calls: Low-level graphics blitting
- Notes: `heightmod` suggests vertical stretching; color parameter enables palette remapping

---

**Trivial helpers summarized under Notes:**
- `GiveExtraMan()`, `GivePoints()`, `GiveLives()` – wrapper functions for player stat increments
- `GetSavedMessage()`, `GetSavedHeader()` – retrieve metadata from save files
- `DrawKills()`, `DrawNumber()`, `DrawGameString()`, `DrawLives()`, `DrawScore()`, `DrawKeys()`, `DrawTime()`, `DrawTriads()`, `DrawStats()`, `DrawBarHealth()`, `DrawBarAmmo()` – individual HUD element renderers
- `GM_UpdateBonus()`, `GM_DrawBonus()`, `ScreenShake()`, `DoBorderShifts()` – frame effects
- `DrawEpisodeLevel()`, `DrawPause()`, `DrawPauseXY()` – state-specific UI overlays
- `DoLoadGameAction()`, `GetLevel()`, `ClearTriads()` – miscellaneous state helpers

## Control Flow Notes
**Init:** `SetupPlayScreen()` prepares the HUD layout; `SD_PreCache()` loads graphics.

**Frame:** `DrawPlayScreen()` renders all HUD elements each tic; `Update*` functions modify state; damage/healing respond to gameplay events.

**Progression:** `LevelCompleted()` / `Died()` / `BattleLevelCompleted()` trigger state transitions and score updates.

**Save/Load:** `SaveTheGame()` / `LoadTheGame()` persist state to disk; `CheckHighScore()` updates leaderboard on level end.

## External Dependencies
- **rt_actor.h**: `objtype` (actor/entity structure), `AlternateInformation`, `exit_t` (level exit codes), enemy and weapon class definitions
- **lumpy.h**: `pic_t` (picture/sprite header)
- **rt_cfg.h**: `AlternateInformation` (resource path/availability)
- **rt_playr.h**: `playertype` (player state structure)
- **Other:** Sound and rendering systems (e.g., `GameMemToScreen()` implies framebuffer/palette API)
