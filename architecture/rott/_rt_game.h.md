# rott/_rt_game.h

## File Purpose
Private header for RT_GAME.C that defines constants and internal function declarations for game rendering, HUD layout, and multiplayer features. Centralizes UI element positioning and provides weapon/gameplay macros specific to Rise of the Triad.

## Core Responsibilities
- Define HUD element positioning constants (kills, players, health, ammo, score, keys, power, armor, lives, timer)
- Declare the `STR` struct for internal string handling
- Provide weapon classification macros (`WEAPON_IS_MAGICAL`) with shareware/full game branching
- Declare private rendering functions (multiplayer pic drawing, high score display, memory-to-screen blits)
- Define save game constraints and other game-specific limits

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| STR | struct | Fixed-size string buffer (10 bytes) with explicit length field |

## Global / File-Static State
None.

## Key Functions / Methods

### DrawMPPic
- Signature: `void DrawMPPic(int xpos, int ypos, int width, int height, int heightmod, byte *src, boolean bufferofsonly)`
- Purpose: Render a multiplayer-related picture (likely player portrait or team indicator) to screen
- Inputs: Position (x, y), dimensions (width, height), height modifier, source pixel buffer, bufferofsonly flag
- Outputs/Return: None (direct screen draw)
- Side effects: Modifies screen framebuffer; memory read from `src`
- Calls: Not visible in this file
- Notes: Parameter `heightmod` suggests variable-height scaling; `bufferofsonly` likely controls buffer selection

### DrawHighScores
- Signature: `void DrawHighScores(void)`
- Purpose: Render the high score display/leaderboard to screen
- Inputs: None (reads global game state)
- Outputs/Return: None
- Side effects: Modifies screen framebuffer
- Calls: Not visible in this file
- Notes: Called at end of game or from menus; requires pre-loaded high score data

### GM_MemToScreen
- Signature: `void GM_MemToScreen(byte *source, int width, int height, int x, int y)`
- Purpose: Copy a rectangular region from system memory to screen framebuffer at specified position
- Inputs: Source pixel buffer, region dimensions, destination screen coordinates
- Outputs/Return: None
- Side effects: Direct screen framebuffer modification; potential memory bandwidth I/O
- Calls: Not visible in this file
- Notes: Generic graphics blit; likely used for HUD sprite/image drawing

## Control Flow Notes
This header is consumed by RT_GAME.C during the **render/HUD** phase of each frame. The positioning constants define a fixed 320×200 (or similar) screen layout with:
- Top row: score, keys, power, armor, men, time, game time, lives, triad
- Mid-left: kills leaderboard (up to 10 players)
- Mid-right: players/teamplay roster
- Bottom: health and ammo bars
- Overlay: leader banner at top

The three function prototypes are internal render calls invoked during game state visualization (end-of-match stats, in-game HUD, multiplayer display).

## External Dependencies
- Conditionally compiled: `SHAREWARE` macro (defines weapon classification differently for commercial vs. freeware)
- External symbols used: `gamestate.teamplay` (game state manager), `byte`, `boolean` (primitive types, defined elsewhere)
- Implicit dependencies: graphics system, framebuffer, high score persistence system
