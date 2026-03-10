# rott/_engine.h
## File Purpose
Private header file that defines internal utility macros for the rendering engine. Contains conditional compilation guards and helper macros for spatial comparisons and sign calculation.

## Core Responsibilities
- Define private engine-internal macros
- Provide tile comparison logic for wall rendering or spatial checks
- Provide sign function for directional calculations

## External Dependencies
- `posts` (defined elsewhere): global rendering primitive array
- No #include directives; intended for inclusion in engine implementation files

# rott/_isr.h
## File Purpose
Defines interrupt service routine (ISR) constants for x86 interrupt vector assignments. Provides symbolic names for hardware interrupt numbers used by the game engine's interrupt handlers.

## Core Responsibilities
- Define interrupt vector constants for system-level event handling
- Provide ISR-related macro definitions for cross-module consistency
- Support DOS/early Windows interrupt architecture used by the game

## External Dependencies
- `_isr_private` header guard (file-private; prevents multiple inclusion)
- No external includes or runtime dependencies

# rott/_rt_acto.h
## File Purpose
Private header for the actor/character system defining constants, macros, and data structures for managing game actors (enemies, projectiles, props). Implements state machines, physics parameters, and collision/behavior utilities for Rise of the Triad.

## Core Responsibilities
- Define actor state machine (11 states: STAND, PATH, COLLIDE, CHASE, AIM, DIE, FIRE, WAIT, CRUSH, etc.)
- Provide physics constants (speeds, momentum scales, fall/rise clipping planes)
- Define actor behavioral flags (PLEADING, EYEBALL, UNDEAD, attack modes)
- Implement inline macros for collision checks, direction parsing, state transitions
- Define saved actor structures for serialization
- Prototype core actor behavior functions

## External Dependencies
- **Actor physics:** References `ParseMomentum()`, `ActorMovement()`, `NewState()` (defined elsewhere)
- **Collision/doors:** `LinkedOpenDoor()`, `doorobjlist[]`, `WallCheck()`
- **Sound:** `SD_PlaySoundRTP()`, BAS actor sound table
- **AI pathing:** `dirangle8[]`, `dirorder[]`, `dirdiff[]` direction lookup tables
- **Random:** `GameRandomNumber()`
- **Rendering:** Shape offset arrays, sprite state definitions
- **Game state:** `gamestate.violence`, `MISCVARS` global timer block

# rott/_rt_buil.h
## File Purpose
Private header file for Rise of the Triad's build/rendering subsystem. Defines compile-time constants and data structures for managing textured planes, menu layout, and viewport rendering parameters.

## Core Responsibilities
- Define texture rendering constants (dimensions, scaling parameters)
- Specify menu UI layout constants (offsets, title positioning)
- Declare the `plane_t` structure for representing renderable geometric planes
- Provide utility macros (MAX, MAXPLANES)
- Configure fixed-resolution rendering pipeline parameters

## External Dependencies
- Included by runtime build/rendering modules (not inferable from this file)
- No external includes or dependencies visible


# rott/_rt_com.h
## File Purpose
Private header file defining network synchronization packet structures and constants for multiplayer/networked gameplay. Establishes the protocol for synchronizing game state between networked instances using a multi-phase handshake.

## Core Responsibilities
- Define synchronization packet structure and format
- Establish phase constants for multi-stage synchronization protocol
- Define timing and sizing parameters for sync operations
- Provide metadata wrapper around sync packets

## External Dependencies
- **C standard types**: `byte`, `int` (language primitives; assume standard integer definitions elsewhere in codebase)
- No explicit includes or external symbol dependencies visible in this file

# rott/_rt_dman.h
## File Purpose
Header file defining constants for audio recording and playback buffer management. Likely used by the sound/music system to configure real-time digital audio handling.

## Core Responsibilities
- Define recording sample rate (7 kHz)
- Configure recording buffer size (16 KB)
- Configure playback buffer size (16 KB)
- Configure playback delta/chunk size for streaming (256 bytes)

## External Dependencies
None; standard C preprocessor guards only.

---

**Summary:** Minimal configuration header for audio I/O parameters. The constants suggest a low-fidelity 7 kHz recording/playback system typical of early 1990s DOS/retro game audio. Files including this are responsible for allocating and managing buffers per these sizes.

# rott/_rt_door.h
## File Purpose
Header file defining door and touch plate mechanics for the game engine. Contains structure definitions and timing/flag constants for door opening animations, push walls, and touch plate trigger systems.

## Core Responsibilities
- Define touch plate state structure (`saved_touch_type`) with timing and action tracking
- Define timing constants for door open/close animations (OPENTICS = 165 tics)
- Define flag constants for door state marking in the level
- Define push wall animation parameters (frame count and tic timing)

## External Dependencies
None—self-contained header with no external includes or symbol dependencies.

# rott/_rt_draw.h
## File Purpose
Private header for the rendering subsystem, defining constants, macros, and function declarations for drawing player weapons, transforming 3D geometry, and managing sprite rendering and lighting in the game's frame loop.

## Core Responsibilities
- Define rendering configuration constants (Z-buffer limits, visibility thresholds, height scaling factors)
- Declare weapon drawing and sprite rendering functions
- Declare geometric transformation functions (plane transformation, rotation calculation)
- Define screensaver state structure for menu animation
- Conditionally define weapon graphics count based on build variant (shareware vs. full)

## External Dependencies
- **develop.h**: Build configuration flags (SHAREWARE, TEXTMENUS, etc.) controlling conditional compilation
- **Implicit types** (defined elsewhere):
  - `objtype`: game object structure
  - `visobj_t`: visible/renderable object structure
  - `boolean`: boolean type (likely typedef'd)

# rott/_rt_film.h
## File Purpose
Defines structures and constants for a film/demo/cutscene playback system. Supports timestamped events with visual properties and actor state tracking across sequential frames.

## Core Responsibilities
- Define event type enumerations (backgrounds, sprites, palettes, fades)
- Specify event properties (position, scale, velocity, animation length)
- Track actor state during film playback (current position/scale, event index)
- Set capacity limits for events and actors per film sequence

## External Dependencies
Standard C types only; no external includes visible.

# rott/_rt_floo.h
## File Purpose
Private header file defining compile-time constants for floor and sky rendering in the ray-traced renderer. Establishes limits on view dimensions, sky segment geometry, and rendering thresholds.

## Core Responsibilities
- Define maximum view height constant (linked to global screen height)
- Set hard limit on sky segment count (geometry optimization)
- Establish sky data structure size constraint
- Define minimum sky height threshold for rendering decisions

## External Dependencies
- `MAXSCREENHEIGHT` (defined elsewhere) — referenced by MAXVIEWHEIGHT macro, indicating coupling to global screen/viewport configuration
- Standard C preprocessor (no function library includes)

# rott/_rt_game.h
## File Purpose
Private header for RT_GAME.C that defines constants and internal function declarations for game rendering, HUD layout, and multiplayer features. Centralizes UI element positioning and provides weapon/gameplay macros specific to Rise of the Triad.

## Core Responsibilities
- Define HUD element positioning constants (kills, players, health, ammo, score, keys, power, armor, lives, timer)
- Declare the `STR` struct for internal string handling
- Provide weapon classification macros (`WEAPON_IS_MAGICAL`) with shareware/full game branching
- Declare private rendering functions (multiplayer pic drawing, high score display, memory-to-screen blits)
- Define save game constraints and other game-specific limits

## External Dependencies
- Conditionally compiled: `SHAREWARE` macro (defines weapon classification differently for commercial vs. freeware)
- External symbols used: `gamestate.teamplay` (game state manager), `byte`, `boolean` (primitive types, defined elsewhere)
- Implicit dependencies: graphics system, framebuffer, high score persistence system

# rott/_rt_in.h
## File Purpose
Private header for RT_IN.C input handling module. Defines low-level input device constants, ISR numbers, and an inline x86 assembly function for BIOS mouse interrupt control in a DOS/legacy environment.

## Core Responsibilities
- Define ISR and BIOS interrupt numbers for keyboard, mouse, and joystick input
- Specify hardware control constants for mouse subcommands and joystick scaling
- Declare and implement low-level mouse control via inline x86 assembly

## External Dependencies
- No local includes
- External: x86 BIOS interrupts (INT 9 for keyboard, INT 0x33 for mouse)
- Assumes: Watcom C compiler, real-mode x86 architecture, DOS environment

# rott/_rt_main.h
## File Purpose
Private header for the main game loop subsystem. Declares core loop functions (`GameLoop`, `PlayLoop`), keyboard polling, screen capture utilities (LBM/PCX writers), and related bitmap format structures for the ROTT engine.

## Core Responsibilities
- Game loop initialization and execution entry points
- Keyboard input polling interface
- Color palette management
- Screen-to-file capture (LBM and PCX formats) with conditional compilation
- Bitmap file format definitions (BMHD/IFF, PCX)
- Quick-load state checking
- Time constants for quit behavior

## External Dependencies
- **Includes**: `develop.h` (feature/debug flags)
- **Types used**: `void`, `boolean`, `byte`, `char`, `int`, `unsigned char`, `unsigned short`, `short` (all C primitives; definitions elsewhere)
- **Macros**: `QUITTIMEINTERVAL`, `SAVE_SCREEN` (conditional compilation flag)
- **Symbols defined elsewhere**: `develop.h` defines all feature flags (`SHAREWARE`, `SUPERROTT`, etc.) and debug modes

# rott/_rt_map.h
## File Purpose
Private header for map rendering utilities in the ROTT game engine. Declares an optimized memory-fill function and defines color constants for rendering different map elements (walls, doors, actors, sprites, sky).

## Core Responsibilities
- Define color palette indices for map visualization (`MAP_*COLOR` macros)
- Declare `FastFill()`, an optimized x86 inline-assembly function for rapid memory initialization
- Provide map scaling factor (`FULLMAP_SCALE`)

## External Dependencies
- **Compiler directive:** `#pragma aux` (Watcom C++ x86 inline assembly)
- No external symbols; self-contained utilities.

# rott/_rt_menu.h
## File Purpose
Private header for the menu system (RT_MENU.C). Defines all UI layout constants, color palette indices, keyboard scan code lookup tables, and function prototypes for the game's menu screens (main menu, settings, load/save, controls configuration, difficulty selection, multiplayer options).

## Core Responsibilities
- Define color constants for menu rendering (borders, backgrounds, active/inactive states)
- Specify screen layout dimensions and positioning for all menu panels (main, controls, sound, load/save, keyboard configuration)
- Provide keyboard scan code name lookups (both standard and extended keys)
- Declare all menu handler and drawing functions
- Define input control structures and enumeration types
- Store game configuration constants (max custom levels, save slots, etc.)

## External Dependencies
- **rt_in.h** – Input system types: `ScanCode`, `ControlType`, `KeyboardDef`, `JoystickDef`, mouse/joystick hardware state globals
- Undefined external types: `boolean`, `byte`, `word` (standard C typedef aliases, likely from a common header)
- Rendering functions called but not declared: implied to exist in menu implementation (RT_MENU.C)


# rott/_rt_msg.h
## File Purpose
Private header file that defines timing constants for the message display system. Provides a single timing parameter used across the engine for message duration management.

## Core Responsibilities
- Define message display timing constant

## External Dependencies
None—self-contained constant definition with only copyright headers.

# rott/_rt_net.h
## File Purpose
Private header for network packet and command synchronization infrastructure in multiplayer RoTT. Defines macros for accessing command buffers, packet addressing, network timeouts, and declares packet handling functions and status enums.

## Core Responsibilities
- Define macro accessors for player/client/server command buffers indexed by time
- Define packet addressing and command state lookup macros
- Specify network timeout constants (standard, modem, server)
- Declare packet lifecycle functions (prepare, send, receive, resend, process)
- Declare status checking and synchronization functions
- Define enums for setup states, command status, and player presence

## External Dependencies
- **Includes**: None visible (private header structure only)
- **External types**: `MoveType` (packet data), `COM_ServerHeaderType` (server header)
- **External symbols**: 
  - Global arrays: `PlayerCmds`, `ClientCmds`, `LocalCmds`, `ServerCmds`, `CommandState`
  - Timing: `controlupdatestartedtime`, `controlupdatetime`, `serverupdatetime`, `VBLCOUNTER` (vertical blank counter)
  - Constants: `MAXCMDS` (command ring buffer size)

# rott/_rt_play.h
## File Purpose
Private header defining constants, macros, and forward declarations for player gameplay mechanics in Rise of the Triad. Covers player movement, weapon handling, camera/horizon control, and VR input mapping.

## Core Responsibilities
- Defines angle limits and speeds for camera tilt, horizon, and player view control
- Provides macros for weapon state transitions (`StartWeaponChange`) and player physics state queries
- Defines movement speed constants (run, base move, jump, thrust)
- Provides distance and angle calculation macros for collision and physics
- Maps VR controller buttons to gameplay actions (0–15)
- Declares forward signatures for enemy attack functions (bat, dog)

## External Dependencies
- **Include:** `watcom.h` (fixed-point math: `FixedMulShift`).
- **Undefined symbols** (defined elsewhere):
  - `FINEANGLES`, `ANG180` — Angle quantization constants.
  - `WEAPONS[]` — Weapon table (screen height, etc.).
  - `SD_PlaySoundRTP()`, `SD_SELECTWPNSND` — Sound API.
  - `player`, `ob`, `pstate` — Global/parameter game objects.
  - `SHOW_BOTTOM_STATUS_BAR()`, `DrawBarAmmo()` — UI functions.

# rott/_rt_rand.h
## File Purpose

Private header file for the random number generator module. Provides a precomputed lookup table of 2048 pseudo-random byte values used by the game engine's RNG implementation in `rt_rand.c`.

## Core Responsibilities

- Define the size constant for the random table
- Supply a precomputed 2048-byte lookup table of pseudo-random values
- Enable deterministic, fast random number generation without runtime seeding

## External Dependencies

- None — self-contained data structure
- Intended for inclusion by `rt_rand.c` only (marked as "private header")

# rott/_rt_scal.h
## File Purpose
Header file defining scaling and sizing constants for the rendering system. Specifically provides the player height constant used in viewport and geometry calculations. Part of the internal rendering architecture (indicated by the "_private" guard naming).

## Core Responsibilities
- Define player height constant for rendering calculations
- Establish the scaling basis for player-relative rendering
- Provide a single point of definition for geometry constants

## External Dependencies
- `HEIGHTFRACTION` – defined elsewhere (likely in a related scaling header or math constants file)

# rott/_rt_ser.h
## File Purpose
Private header defining UART 8250 serial port register offsets, control flags, and a circular queue structure for buffering serial data. Declares low-level serial I/O functions and CPU interrupt control macros used for modem-based network communication in the game.

## Core Responsibilities
- Define 8250 UART register addresses and bit flags for interrupt, FIFO, line control, modem status
- Define circular queue structure (`que_t`) for buffering transmitted/received serial data
- Declare serial port lifecycle functions (init, shutdown, enable)
- Declare byte-level read/write functions and interrupt service routine
- Provide low-level I/O macros for port access and CPU interrupt control

## External Dependencies
- **Include**: `rottnet.h` (defines `MAXPACKETSIZE`, `MAXCOMBUFFERSIZE`, `rottcom_t` structure)
- **Defined elsewhere**: 
  - `inp()`, `outp()` — low-level port I/O (DOS/x86 intrinsics)
  - `_disable()`, `_enable()` — CPU interrupt control (Watcom/DOS intrinsics)
  - Interrupt vector setup and EOI handling (assumed in ISR implementation)

# rott/_rt_soun.h
## File Purpose
Private header defining internal structures, flags, and constants for the Rise of the Triad sound system. Manages active sound instances, music data, and priority/behavior configuration for audio playback.

## Core Responsibilities
- Define `sound_t` structure for tracking active sound instances
- Define `song_t` structure for music/song metadata
- Enumerate 17 priority levels for sound playback arbitration
- Provide sound behavior flags (looping, overwrite, pitch shift, etc.)
- Map sound types to priority constants (e.g., explosions, weapon fire, enemies, UI)
- Declare sound playback and validation functions
- Expose tuning constants (distance shift, random pitch shift)

## External Dependencies
- `sounds[]` array (macro `SoundOffset` references; defined elsewhere)
- `soundtype` variable (used in `SoundOffset` macro)
- `RandomNumber()` function (used in `PitchOffset` macro; defined elsewhere)
- `USEADLIB` constant (= 255; likely device/driver enum)
- `GUSMIDIINIFILE` path reference

**Trivial helpers:**
- Macros: `PitchOffset()` (random pitch ±128), `SoundOffset(x)` (retrieve sound handle), `SD_DISTANCESHIFT`, `SD_RANDOMSHIFT` (tuning constants)
- Flags: `SD_OVERWRITE`, `SD_WRITE`, `SD_LOOP`, `SD_PITCHSHIFTOFF`, `SD_PLAYONCE` (5 bits for sound behavior)

# rott/_rt_spba.h
## File Purpose
Private header for spaceball input device support in ROTT. Defines the sign-extraction macro and the count of spaceball buttons (6 buttons on the hardware device).

## Core Responsibilities
- Define the `SGN()` macro for extracting the sign of a numeric value (1 if positive, -1 if negative)
- Declare the spaceball button count as a compile-time constant
- Serve as a private include guard to prevent redefinition of spaceball-related constants

## External Dependencies
- No external includes or symbols; this is self-contained.
- The `_rt_spba_private` guard suggests it is private to the spaceball subsystem and should not be included by external code.

# rott/_rt_stat.h
## File Purpose
Private header file declaring functions and types for managing static objects (doors, walls, decorations, lights) in the ROTT game engine. Provides data structures for animated wall info and saved static state, plus utility macros for fire color animation and light detection.

## Core Responsibilities
- Declare functions for adding and managing static/animated static objects
- Define types for persisting static object state (saved_stat_type)
- Declare precaching function for static object sounds
- Provide animation timing constants and light detection utility

## External Dependencies
- `statobj_t` – object type (defined elsewhere)
- `sprites` global array – 2D sprite lookup (defined elsewhere)
- Standard C types: int, char, byte, short int, signed char

# rott/_rt_str.h
## File Purpose
Private header for string drawing and measurement utilities in the rendering system. Declares function prototypes for proportional string operations and initializes global function pointers used throughout the engine for text rendering.

## Core Responsibilities
- Declare prototypes for proportional string drawing (`VWB_DrawPropString`)
- Declare prototypes for proportional string measurement (`VW_MeasurePropString`)
- Define and initialize global function pointers for abstracted string operations (`USL_MeasureString`, `USL_DrawString`)
- Provide a layer of indirection for string rendering across the engine

## External Dependencies
- `font_t` type (defined elsewhere)
- Actual implementations of `VWB_DrawPropString` and `VW_MeasurePropString` (RT_STR.C)
- Video buffer interface (implicit in `VWB_*` naming convention)

# rott/_rt_swft.h
## File Purpose
Private header for RT_SWIFT.C providing declarations, defines, and structures for the SWIFT input device driver. Manages DOS memory allocation, DPMI real-mode interrupt handling, and device state (Cyberman joystick support).

## Core Responsibilities
- Define interrupt codes (DPMI `0x31`, mouse `0x33`) and device type constants
- Provide x86 register access macros (`AX`, `BX`, `CX`, etc.) for register operations
- Declare global device state (active flag, attached device type)
- Define DPMI real-mode interrupt structure for interrupt context
- Declare DOS memory management functions (allocate/deallocate)
- Declare mouse interrupt handler entry point

## External Dependencies
- DOS/DPMI-specific: interrupt codes, register structures, real-mode interrupt handling
- x86 inline register macros for CPU register manipulation
- No external symbol dependencies defined in this file

# rott/_rt_ted.h
## File Purpose
Private header defining data structures and constants for the TED level editor and map file format. Provides layout constants for precache UI elements and build-time level filename configuration based on game edition.

## Core Responsibilities
- Define map file format structures (`mapfiletype`, `maptype`, `cachetype`)
- Provide macros for actor type checking at map grid positions
- Define precache display layout constants (strings, progress bars, LEDs)
- Conditionally configure level filenames for shareware vs. registered builds
- Define RLEW compression tags and RTL file format version constants

## External Dependencies
- **Includes:** `rt_actor.h` (actor class definitions), `develop.h` (build flags: `SHAREWARE`, `SUPERROTT`, `SITELICENSE`)
- **Symbols defined elsewhere:** `actorat[][]` (actor grid), `sprites[][]` (sprite grid), `objtype` (actor structure), `statobj_t` (static object)

# rott/_rt_util.h
## File Purpose
A private utility header defining low-level VGA hardware constants, file paths for error/debug logging, and basic utility macros. Used for palette manipulation and debugging infrastructure in the Rise of the Triad engine.

## Core Responsibilities
- Define VGA hardware I/O port addresses for palette (PEL) read/write operations
- Provide file path constants for error logs and debug output
- Supply utility macros for sign determination and color weight calculations
- Define screen position constants for error message placement

## External Dependencies
None—pure preprocessor definitions with no external symbols referenced.

# rott/_rt_vid.h
## File Purpose
Private header for RT_VID.C containing video/graphics utility macros. Provides line-drawing and pixel-manipulation abstractions that wrap lower-level VL_* functions, converting from endpoint notation to start-point + length notation.

## Core Responsibilities
- Define pixel-to-block conversion constant (PIXTOBLOCK)
- Provide horizontal line drawing macro wrappers (VW_Hlin, VW_THlin)
- Provide vertical line drawing macro wrappers (VW_Vlin, VW_TVlin)
- Abstract away coordinate calculation logic from call sites

## External Dependencies
- Assumes VL_Hlin, VL_Vlin, VL_THlin, VL_TVlin are defined elsewhere (likely in VID-related modules)
- No explicit includes in this file
- Include guard `_rt_vid_private` prevents multiple inclusion

# rott/_st_menu.h
## File Purpose
Private header for the menu subsystem (st_menu.c). Defines constants, data structures, and function prototypes for rendering and handling the game's UI menus, including main menu, modem setup, music/sound configuration, and serial port selection.

## Core Responsibilities
- Define menu layout constants (colors, screen positions, dimensions)
- Declare the `EditList` structure for storing phonebook/configuration entries
- Declare all menu rendering functions (draw, erase, position cursor, display info)
- Declare all menu handler functions (input, selection, validation)
- Define static menu data structures for main menu, sub-menus, and configuration dialogs
- Provide menu-specific handlers for modular features (modem, serial, music, FX, etc.)

## External Dependencies
- **`#include "rt_menu.h"`**: Provides base types:
  - `CP_iteminfo`: Menu layout metadata (x, y, item count, current position, indent level, font size)
  - `CP_itemtype`: Menu item (active flag, texture/string, hotkey letter, callback function pointer)
- **Inferred types** (defined elsewhere in rt_menu.h):
  - `CP_MenuNames` (char pointer array)
  - `menuptr` (function pointer typedef)
  - Globals: `colorname[]`, `NewGame`, `pickquick`, font pointers, window/buffer coords, game state flags

---

**Notes on Legacy Features:** The modem, serial port, and device selection menus reflect 1990s multiplayer infrastructure (dial-up networking, soundcard configuration). Modern builds likely disable or stub these.

# rott/_w_wad.h
## File Purpose
Private header defining the WAD file format structures and constants. WAD files are the game's resource containers (lumps) used for maps, sprites, textures, and other assets. Provides structures for parsing WAD headers and lump directory entries.

## Core Responsibilities
- Define WAD file header structure (`wadinfo_t`) for identifying and locating lump directories
- Define in-memory lump metadata structure (`lumpinfo_t`) for tracking loaded resources
- Define WAD file lump directory entry structure (`filelump_t`) for reading from disk
- Provide compile-time WAD checksum validation constants (varies by build config: Shareware/Deluxe/Low-cost)
- Specify lump check period for periodic validation

## External Dependencies
- `#include "develop.h"` — provides build configuration macros (`SHAREWARE`, `DELUXE`, `LOWCOST`, `SUPERROTT`) that determine WAD checksum at compile time

# rott/_z_zone.h
## File Purpose
Private header defining the core data structures for the ROTT engine's zone-based memory allocator. Implements a custom heap with linked-list block management, supporting memory corruption detection for debugging.

## Core Responsibilities
- Define `memblock_t` structure for tracking allocated/free memory blocks in a doubly-linked list
- Define `memzone_t` structure for representing a memory pool with aggregate tracking
- Provide constants for memory management (fragment size, memory limits, DPMI interrupt)
- Support optional memory corruption detection via pre/post tags

## External Dependencies
- `develop.h` — provides feature flags (`MEMORYCORRUPTIONTEST`) that conditionally include debug tags in `memblock_t`


# rott/cin_actr.c
## File Purpose
Manages a doubly-linked list of cinematic actors that encapsulate visual effects during cinematic sequences. Each actor holds a type and opaque effect data pointer. Provides lifecycle management (creation, deletion, update, rendering) for actors within the cinematic system.

## Core Responsibilities
- Maintain a global linked list of active cinematic actors (head/tail pointers)
- Allocate and deallocate actor objects with bounds checking
- Insert actors into the linked list and remove them with proper pointer bookkeeping
- Initialize and shutdown the cinematic actor system (state reset)
- Spawn new actors with specific effect types and data payloads
- Update all actors each frame, removing those marked as complete
- Render all actors in layered phases (screen functions → background → sprites → backdrop → foreground → palette)

## External Dependencies
- **Headers:** `cin_glob.h`, `cin_def.h`, `cin_actr.h`, `cin_efct.h`, `modexlib.h`, `memcheck.h`
- **Defined elsewhere:**
  - `SafeMalloc()`, `SafeFree()` – memory management from memcheck.h
  - `UpdateCinematicEffect()`, `DrawCinematicEffect()` – effect handlers from cin_efct.c
  - `XFlipPage()` – video/display update from modexlib.c
  - `Error()` – error reporting (stdio or engine)
  - `MAXCINEMATICACTORS` constant (30, from cin_def.h)
  - `enum_eventtype`, `actortype` – from cin_def.h

# rott/cin_actr.h
## File Purpose
Public header for cinematic actor management. Declares the API for managing, rendering, and updating actors within cinematic sequences (animated sprites, backdrops, and effects displayed during game cutscenes).

## Core Responsibilities
- Maintain a linked list of active cinematic actors (via `firstcinematicactor` and `lastcinematicactor`)
- Add and remove actors from the cinematic actor list
- Allocate new cinematic actor instances
- Initialize and shut down the cinematic actor subsystem
- Spawn actors from cinematic event data
- Update and render all active cinematic actors each frame

## External Dependencies
- **Includes**: `cin_glob.h` (cinematic timing macros), `cin_def.h` (type definitions)
- **Types used**: `actortype`, `enum_eventtype` (defined in cin_def.h)
- **Macros used**: Not visible in header
- **Defined elsewhere**: `actortype` structure, `enum_eventtype` enumeration, underlying memory allocation and rendering functions

# rott/cin_def.h
## File Purpose
Header file defining data structures and constants for the cinematic sequence system. Provides linked-list based event scheduling, effect types (backgrounds, sprites, palettes, FLI animations), and fixed-point math definitions for smooth animation interpolation.

## Core Responsibilities
- Define cinematic event enumeration (backgrounds, sprites, backdrops, palettes, fade, FLI, etc.)
- Define linked-list node types for events and actors in the cinematic timeline
- Define effect-specific parameter structures (sprites, backdrops, FLI animations, palettes)
- Provide fixed-point arithmetic constants for sub-pixel-precision transformations
- Define capacity limits for cinematic events and actors

## External Dependencies
- Standard C types: `int`, `char`, `byte`, `boolean` (defined elsewhere in project)

# rott/cin_efct.c
## File Purpose
Implements cinematic effects rendering for the Rise of the Triad engine. Handles creation, updating, and drawing of interactive cutscene elements: animated sprites, scrolling backgrounds, palette changes, and fade transitions. Provides the core visual pipeline for in-game cinematics.

## Core Responsibilities
- Create and initialize cinematic event objects (sprites, backgrounds, flics, palettes)
- Draw scaled sprites and background layers using column-based fixed-point rendering
- Manage cinematic state updates (animation frames, position interpolation, duration countdown)
- Precache/load cinematic assets from WAD lumps into memory
- Dispatch rendering and update calls based on effect type enum
- Implement fade-to-black and palette transition effects
- Handle VGA plane-based rendering for 320×200 mode

## External Dependencies
- **Cinematic system headers:** `cin_glob.h` (timing), `cin_util.h` (palette), `cin_def.h` (types), `cin_main.h` (extern `cinematicdone`)
- **Graphics/rendering:** `f_scale.h` (scaled column rendering globals and `R_DrawFilmColumn`), `modexlib.h` (VGA mode functions)
- **WAD/lump system:** `w_wad.h` (`W_CacheLump*`, `W_GetNumForName`), `lumpy.h` (patch/lpic structs)
- **Memory:** `z_zone.h` (`SafeMalloc`, `SafeFree`)
- **Format support:** `fli_glob.h` (FLI video)
- **Compiler/debug:** `watcom.h`, `memcheck.h`
- **Standard C:** `string.h` (`strcpy`, `memcpy`), `conio.h` (console I/O)

**Defined elsewhere:**
- `SafeMalloc`, `SafeFree` - memory allocator
- `W_CacheLumpName`, `W_CacheLumpNum`, `W_GetNumForName` - WAD cache
- `CinematicGetPalette`, `CinematicSetPalette` - palette I/O
- `CinematicDelay`, `GetCinematicTics` - timing
- `VL_SetVGAPlaneMode`, `VL_ClearVideo`, `XFlipPage` - VGA control
- `R_DrawFilmColumn`, `DrawFilmPost`, `FixedMul` - low-level rendering
- `Error` - error reporting
- VGA macros: `VGAWRITEMAP`, `VGAMAPMASK`, `bufferofs`, `ylookup`

# rott/cin_efct.h
## File Purpose
Public header declaring the cinematic effect system interface. Provides factory functions to spawn cinematic events (animations, sprites, backgrounds, palettes) and drawing/update routines to render and advance them during playback. This is the primary API for sequencing cutscenes.

## Core Responsibilities
- Spawn cinematic effect objects (FLICs, sprites, backgrounds, palettes)
- Draw/render active cinematic effects to screen
- Update cinematic effect state each frame (animation frames, sprite positions, scrolling)
- Generic effect dispatch (DrawCinematicEffect, UpdateCinematicEffect) for polymorphic handling
- Precache effect resources before playback
- Manage buffer clearing and screen state
- Performance profiling support

## External Dependencies
- `cin_glob.h`: Cinematic timing (CinematicDelay, GetCinematicTime) and abort control
- `cin_def.h`: Type definitions (enums, structs)
- Transitive: `rt_def.h`, `rt_util.h`, `isr.h`, `<time.h>` (from cin_glob.h)
- Graphics/rendering system (not visible in this file; callee of Draw* functions)
- WAD resource system (lumpnum references in DrawPostPic)

# rott/cin_evnt.c
## File Purpose
Manages the lifecycle of cinematic events: creating, storing, parsing, and processing time-based events for cutscenes. Events are stored in a doubly-linked list and triggered by the update loop when the current playback time matches their scheduled time.

## Core Responsibilities
- Maintain a doubly-linked list of cinematic events (add, delete, create)
- Parse cinematic event definitions from script tokens
- Trigger events at their scheduled times during playback
- Pre-cache resources for all events in a cinematic
- Manage system startup and shutdown

## External Dependencies
- **cin_glob.h, cin_efct.h, cin_actr.h, cin_def.h:** Cinematic system definitions and effect/actor spawning.
- **scriplib.h:** Script parsing (`GetToken`, `ParseNum`).
- **w_wad.h:** WAD resource caching (`W_CacheLumpName`); graphics types (lpic_t, patch_t).
- **z_zone.h:** Memory management (`SafeMalloc`, `SafeFree`).
- **string.h:** C standard string functions (`strcpy`, `strcmpi`).
- **memcheck.h:** Memory debugging.

**Security note:** Fixed-size buffers (10 bytes) in parse functions with unbounded `strcpy` calls create potential buffer overflows.

# rott/cin_evnt.h
## File Purpose
Public interface for cinematic event management in ROTT. Provides functions to create, manage, and process time-synchronized events (visual effects, sprite animations, backdrop scrolling, palette changes, etc.) during cinematic sequences.

## Core Responsibilities
- Event lifecycle management (create, add, delete, retrieve)
- Maintenance of a doubly-linked event queue (`firstevent`, `lastevent`)
- Time-based event scheduling and execution
- Cinematic initialization and shutdown
- Resource precaching for cinematics

## External Dependencies
- **cin_glob.h**: Global cinematic declarations (`CinematicDelay`, `GetCinematicTime`, `CinematicAbort`)
- **cin_def.h**: Type definitions (`eventtype`, `enum_eventtype`, effect structures: `flicevent`, `spriteevent`, `backevent`, `paletteevent`)
- Indirect: `rt_def.h`, `rt_util.h`, `isr.h`, `<time.h>` (via cin_glob.h)

# rott/cin_glob.c
## File Purpose
Provides a simple abstraction layer for controlling cinematic/cutscene playback. Wraps timing and input functions to allow cinematics to synchronize with the engine's tick system and detect user-initiated abort/skip requests.

## Core Responsibilities
- Synchronize cinematic playback with engine timing via tick counting
- Query elapsed cinematic time for audio/visual synchronization
- Detect user input indicating cinematic should be skipped/aborted
- Clear input acknowledgment state after processing abort requests

## External Dependencies
- **rt_draw.h / rt_draw.c:** `CalcTics()` function; `ticcount` global variable (extern int)
- **rt_in.h / rt_in.c:** `IN_CheckAck()`, `IN_StartAck()` functions
- **cin_glob.h:** Function declarations
- **memcheck.h:** Memory debugging utility (passive inclusion)

# rott/cin_glob.h
## File Purpose
Public interface for cinematic (cutscene) timing and control. Provides functions to synchronize cinematic playback with the VBlank timer, query elapsed time, and handle user abort requests.

## Core Responsibilities
- Declare cinematic delay/synchronization function
- Expose cinematic elapsed time query
- Manage cinematic abort flag (check and clear)
- Define timing macro (`CLOCKSPEED`) based on VBlank counter

## External Dependencies
- **rt_def.h** — provides `boolean` type, general engine constants
- **rt_util.h** — included (purpose not immediately apparent from declarations here)
- **isr.h** — provides `VBLCOUNTER` (35 Hz timer tick constant); declares ISR state (`ticcount`, keyboard queue)
- **<time.h>** — standard C time header (included but usage not inferable here)

# rott/cin_main.c
## File Purpose
Core cinematic playback engine for ROTT. Orchestrates loading, parsing, and executing cinematic scripts; manages timing, frame updates, and rendering for in-game cinematics.

## Core Responsibilities
- Load and parse cinematic script files from disk or WAD lumps
- Manage cinematic timing, frame deltas, and tics-per-frame calculation
- Coordinate startup/shutdown of cinematic subsystems (events, actors, effects)
- Execute main cinematic playback loop with event/actor updates
- Profile rendering performance to calibrate frame timing

## External Dependencies
- **Cinematic subsystems:** cin_glob.h (timing), cin_actr.h, cin_evnt.h, cin_efct.h (update/render)
- **Scripting:** scriplib.h (script parsing globals and functions)
- **WAD system:** w_wad.h (W_GetNumForName, W_CacheLumpNum, W_LumpLength)
- **Graphics:** modexlib.h, lumpy.h (ProfileDisplay)
- **Memory:** z_zone.h (PU_CACHE tag for lump caching)
- **Standard C:** stdio.h, stdlib.h, string.h, conio.h, dos.h, malloc.h, fcntl.h, io.h

# rott/cin_main.h
## File Purpose
Public API for the cinematic/movie playback system in ROTT. Declares functions to load and play movie scripts, manage cinematic timing, and track playback state through a global completion flag.

## Core Responsibilities
- Load movie script files into memory for playback
- Play loaded cinematic sequences with optional rendering modes
- Update cinematic timing state each frame
- Expose global completion status for cinematics
- Support "lumpy" rendering mode (likely palette-based or chunked rendering)

## External Dependencies
- `cin_glob.h` (local cinematic globals/macros)
- `rt_def.h`, `rt_util.h`, `isr.h` (engine core, included transitively)
- `<time.h>` (standard C timing)
- `boolean` type and `VBLCOUNTER` macro defined elsewhere in engine

# rott/cin_util.c
## File Purpose
Provides utility functions for reading and writing VGA color palettes during cinematic playback. These functions directly access VGA hardware I/O ports to synchronize palette state between the engine and display hardware.

## Core Responsibilities
- Read 8-bit VGA color palette from hardware ports into application memory
- Write 8-bit color palette from application memory to hardware ports
- Handle bit-shift scaling between VGA's 6-bit internal representation and the 8-bit storage format
- Support cinematic sequences that require precise color control

## External Dependencies
- `conio.h` – provides `outp()` (write to I/O port) and `inp()` (read from I/O port)
- `modexlib.h` – defines VGA hardware port constants (`PEL_READ_ADR`, `PEL_WRITE_ADR`, `PEL_DATA`)
- `cin_glob.h` – declares cinematic subsystem declarations and includes
- `memcheck.h` – memory debug/check facilities (included but unused in this file)

# rott/cin_util.h
## File Purpose
Header file providing the public interface for cinematic palette operations. Declares functions to read and write the game's cinematic palette state.

## Core Responsibilities
- Define public API for cinematic palette management
- Expose palette getter/setter functions to the cinematic subsystem

## External Dependencies
- `byte` type (defined elsewhere, likely in a common header like `_types.h`)

# rott/develop.h
## File Purpose
Development and build configuration header for Rise of the Triad. Defines compile-time feature flags to enable/disable debug features, cheats, test modes, and game variants. All flags default to 0 (off) except for production settings.

## Core Responsibilities
- Toggle debug modes (DEBUG, DEVELOPMENT, SOUNDTEST, PRECACHETEST)
- Enable test modes for specific subsystems (ELEVATORTEST, LOADSAVETEST, BATTLECHECK)
- Control cheat codes (WEAPONCHEAT)
- Define game variant (SHAREWARE, SUPERROTT, SITELICENSE)
- Provide debug location tracking macros (wami/waminot)
- Gate UI modes (TEXTMENUS vs. default)

## External Dependencies
- `programlocation` variable (defined elsewhere)—used by conditional wami macro

# rott/engine.c
## File Purpose
Implements the core ray-casting renderer that determines which walls are visible from the player's viewpoint. Uses hierarchical screen-space subdivision and fixed-point grid traversal to efficiently cast rays and populate wall geometry data for rendering.

## Core Responsibilities
- **Ray casting**: Cast rays from player viewpoint through screen pixels to find wall intersections
- **Hierarchical culling**: Use a 4-pixel comb filter with binary subdivision to reduce ray count
- **Tile grid traversal**: Fixed-point arithmetic grid walking to find ray-tile intersections
- **Wall property resolution**: Handle special tile types (doors, windows, animated walls, masked objects) and determine texture coordinates
- **Visibility tracking**: Mark visible areas in spotvis and mapseen arrays for map rendering
- **Texture interpolation**: Linearly interpolate texture and height between nearby cast rays to fill screen gaps

## External Dependencies
- **Includes/Imports**: rt_def.h (constants), watcom.h (fixed-point math), engine.h (wallcast_t typedef), _engine.h (macros), rt_eng.h, rt_draw.h, rt_door.h, rt_stat.h, rt_ted.h, rt_view.h (declarations assumed).
- **Defined elsewhere**:
  - `tilemap[][]` — tile grid data
  - `doorobjlist[]`, `maskobjlist[]` — entity arrays
  - `animwalls[]` — animated wall texture table
  - `spotvis[][]`, `mapseen[][]` — visibility tracking
  - `viewx`, `viewy`, `viewwidth`, `viewsin`, `viewcos` — camera state
  - `c_startx`, `c_starty` — initial ray direction
  - `CalcHeight()` — compute wall height
  - `MakeWideDoorVisible()` — multi-tile door visibility
  - `FixedMul()`, `FixedScale()` — fixed-point arithmetic (watcom.h)
  - `MAPSPOT()` macro — map plane access

# rott/engine.h
## File Purpose
Public interface header for the rendering engine module. Declares the main `Refresh()` rendering function, exports the wall-rendering data structure (`wallcast_t`) and related state, and provides a utility macro for querying the map grid.

## Core Responsibilities
- Declare the primary frame-rendering entry point (`Refresh`)
- Export the wall-casting result array (`posts`) used to draw vertical wall segments
- Track the last rendered camera position (`lasttilex`, `lasttiley`)
- Provide map-grid utility macro (`IsWindow`)

## External Dependencies
- `MAPSPOT` macro (used in `IsWindow()`): defined elsewhere (likely a map/tile header)
- `Refresh()` implementation: engine.c

# rott/f_scale.asm
## File Purpose

Optimized x86 assembly implementation for column-based texture scaling and rendering in a software 3D engine. Provides high-performance vertical pixel-column drawing with fixed-point texture filtering, using self-modifying code for dynamic per-call scaling parameters.

## Core Responsibilities

- Draw scaled texture columns to a framebuffer using fixed-point texture coordinate interpolation
- Dynamically configure scaling factors via runtime code patching
- Process pixels in paired batches to maximize throughput
- Provide post-processing layout for interleaved texture data

## External Dependencies

- **_ylookup** (DWORD array): Lookup table mapping Y screen coordinates to framebuffer byte offsets; indexed by `[_cin_yl * 4]`
- Constants: `SCREENWIDTH = 96`, `SCREENROW = 96`

# rott/f_scale.h
## File Purpose
Header file declaring the public interface for film column scaling and rendering operations. Exports global scaling parameters and two key rendering functions used by the engine's vertical scaling subsystem, likely for drawing stretched/scaled sprite columns.

## Core Responsibilities
- Export global scaling state variables (Y-bounds, scale factor, texture mid-point)
- Declare the core R_DrawFilmColumn function for rendering scaled columns
- Declare the DrawFilmPost function for post-processing film output
- Maintain consistent parameter passing conventions via pragma directives

## External Dependencies
- Standard C types (byte, int, void)
- Watcom C compiler pragmas (`#pragma aux`) for register allocation and calling conventions
- Assumed to be linked with implementation files that use these declarations

# rott/fli_def.h
## File Purpose
Defines the binary structures and constants for the FLIC/FLC animation file format (320×200 FLI and variable-resolution FLC). Includes headers for files, frames, and compression chunks, enabling serialization and deserialization of frame-based animations.

## Core Responsibilities
- Define binary layout of FLIC file header with metadata (dimensions, frame count, timing, creation info)
- Define frame and chunk headers for hierarchical file structure
- Enumerate chunk compression/data types (palette, delta, RLE, literal)
- Provide type identifiers and flags for format variant detection

## External Dependencies
- Project-defined types: `Long`, `Ushort`, `Short`, `Ulong`, `Char` (defined elsewhere, likely 16/32-bit C types)
- Uses `#pragma pack(1)` for byte-aligned binary struct layout


# rott/fli_glob.h
## File Purpose
A header file declaring the interface for playing FLI/FLIC animation files. FLI is a video format commonly used in 1990s games for cutscenes and cinematics.

## Core Responsibilities
- Declares the `PlayFlic` function for animation playback
- Provides flexible media playback interface supporting both file-based and memory-based animation sources

## External Dependencies
- Standard C library (function signature only; details in implementation)
- Implementation expected in `fli_glob.c`
- No includes visible in this header file

# rott/fli_main.c
## File Purpose
Implements the FLI/FLC cinematic file player for the ROTT engine. Provides decompression routines for various FLIC frame chunk types and high-level playback control for displaying animated sequences on-screen.

## Core Responsibilities
- Decompress FLIC chunk types (COLOR_256/64, DELTA_FLC/FLI, BYTE_RUN, LITERAL, BLACK)
- Manage FLIC file I/O from disk or in-memory buffer
- Load and parse FLIC headers and frame structures
- Render decompressed frames to screen via pixel/color operations
- Synchronize playback timing with frame delays
- Control cinematic playback (single-play and looping modes)
- Handle user abort (keyboard input during playback)
- Convert between FLI and FLC formats

## External Dependencies
- **Game engine internals:** `cin_glob.h` (cinematic globals, timing), `rt_def.h`, `rt_util.h`, `isr.h`
- **Standard C:** `errno.h` (I/O errors), `string.h` (memcpy), `io.h` (file I/O)
- **FLIC format definitions:** `fli_type.h` (basic C types), `fli_util.h` (Screen, Machine, I/O, Color), `fli_def.h` (FLIC headers/chunks), `fli_main.h` (Flic struct, error codes, macros)
- **Memory management:** `memcheck.h` (likely debug/leak tracking)
- **External symbols (defined elsewhere):**
  - Screen: `screen_open()`, `screen_close()`, `screen_put_dot()`, `screen_copy_seg()`, `screen_repeat_one()`, `screen_repeat_two()`, `screen_put_colors()`, `screen_put_colors_64()`, `screen_width()`, `screen_height()`
  - Machine/Clock/Key: `machine_open()`, `machine_close()`
  - Cinematic: `GetCinematicTime()`, `CinematicAbort()`, `GetCinematicTime()`
  - I/O: `file_open_to_read()`, `file_read_big_block()`, `big_alloc()`, `big_free()`, `lseek()`, `close()`
  - Error reporting: `Error()` (printf-style logging)

# rott/fli_main.h
## File Purpose
Header file for Flic animation file (.fli/.flc format) reading and playback. Defines the Flic structure, playback control interface, and error codes for managing Flic animations within the game engine.

## Core Responsibilities
- Define `Flic` structure for managing open Flic file state and playback
- Declare file I/O functions (open, close, block reading)
- Declare playback functions (single play, looping, frame advancement)
- Provide error reporting via error codes and string lookup
- Define utility macros for memory operations and array handling

## External Dependencies
- **Defined elsewhere**: `FlicHead` (struct), `Machine` (struct), `Screen` (struct), `MemPtr` (typedef), `Boolean` (typedef), `ErrCode` (typedef)
- **Standard C**: `memset()` (via `ClearMem` macro)
- **Attribution**: Flic reader based on Jim Kent code (1992), adapted for Apogee engine

# rott/fli_type.h
## File Purpose
Portable integer type definitions and platform abstraction for the ROTT engine. Provides standardized type names for cross-compiler compatibility and defines boolean/error code semantics. Based on Jim Kent's original Types.h design.

## Core Responsibilities
- Define portable 8/16/32-bit integer type aliases (Char, Uchar, Short, Ushort, Long, Ulong)
- Standardize Boolean and ErrCode semantic types
- Define FileHandle abstraction
- Supply TRUE/FALSE and error code constants (Success, AError)
- Provide compiler/platform macro overrides (int86, inportb, outportb)

## External Dependencies
None; self-contained type definitions for engine-wide use.

# rott/fli_util.c
## File Purpose
Machine-specific abstraction layer for FLI (Flic) animation playback on DOS/VGA hardware. Provides screen display, palette management, keyboard input, clock timing, and file I/O wrappers to isolate platform dependencies from the FLI decoder and renderer.

## Core Responsibilities
- VGA mode switching (mode 0x13: 320×200×8bpp) and graphics initialization via BIOS interrupts
- Direct pixel writing to video memory with horizontal line clipping and bounds checking
- VGA palette (colormap) setup via hardware port I/O (0x3C8–0x3C9)
- Performance-critical rendering using optimized memcpy/memset for common patterns
- Hardware initialization and shutdown orchestration via Machine abstraction
- Wrappers for large-block memory allocation and file I/O (delegate to SafeMalloc/SafeRead)

## External Dependencies
- **BIOS/Hardware:** `int86()` (INT 0x10 for video, INT 0x1A for timer); `inportb()`, `outportb()` for VGA palette port I/O (0x3C8–0x3C9).
- **Standard C:** `memcpy()`, `memset()`, `<stdlib.h>`, `<mem.h>`, `<dos.h>`, `<bios.h>`.
- **Defined elsewhere:** `SafeMalloc()`, `SafeFree()`, `SafeOpenRead()`, `SafeRead()` (memory and file I/O); `ClearStruct()` macro (fli_main.h); type definitions in fli_type.h.

# rott/fli_util.h
## File Purpose
Header file defining machine-specific abstractions for the FLI animation/graphics system in the ROTT engine. Provides portable interfaces to screen rendering, keyboard input, timing, file I/O, and large memory allocation (>64K blocks) to isolate platform-dependent code from the FLI decoder.

## Core Responsibilities
- Abstract screen/video hardware operations (open, close, drawing pixels, color palette management)
- Abstract keyboard polling and input handling
- Abstract system clock/timing operations
- Abstract large block memory allocation for DOS platform constraints
- Abstract binary file reading operations
- Aggregate machine initialization/shutdown of all peripheral devices

## External Dependencies
- **Type definitions**: `Uchar`, `Ushort`, `Ulong`, `Boolean`, `ErrCode`, `FileHandle` (defined elsewhere, likely common header)
- **Scope**: Portable abstractions; platform-specific implementations exist elsewhere (DOS, other OS versions)

# rott/fx_man.h
## File Purpose

Public header for the effects/sound manager subsystem. Declares the interface for hardware-accelerated sound playback, voice management, and real-time audio control (volume, panning, pitch, 3D positioning, reverb). Supports Sound Blaster and compatible sound cards with both 2D and 3D audio mixing.

## Core Responsibilities

- Sound card hardware initialization and detection (particularly Sound Blaster variants)
- Multi-voice sound mixing and playback with priority-based voice allocation
- Audio effect playback for VOC, WAV, and raw PCM formats with loop support
- Real-time sound parameter control (volume, pan, pitch, frequency, reverb)
- 3D audio positioning (angle/distance-based panning and attenuation)
- Callback-driven completion notification for sound events
- Live audio recording at configurable sample rates
- Demand-feed playback for streaming/procedural audio

## External Dependencies

- **sndcards.h**: Sound card enumeration (`soundcardnames` enum for card type constants)
- Implied: Low-level hardware drivers, mixer libraries (not declared here)

# rott/gmove.h
## File Purpose
Minimal header providing a constant `GMOVE` with value 8. This appears to be a module identifier or flag used elsewhere in the engine.

## Core Responsibilities
- Define the `GMOVE` constant (value: 8)

## External Dependencies
None. Standard C header guard only.


# rott/isr.c
## File Purpose
Manages low-level hardware interrupts for timer and keyboard input on DOS systems. Provides ISR hooks for the system timer (PIT) and keyboard controller, maintaining game tick counts and processing raw scan codes into a keyboard event queue.

## Core Responsibilities
- Timer interrupt handling: increments game tick counter (`ticcount`) at ~35 Hz
- Keyboard interrupt handling: reads scan codes, populates keyboard queue, manages shift/extended key states
- Keyboard LED control (caps lock, num lock, scroll lock) via keyboard controller commands
- System timer initialization/shutdown with task-based scheduling fallback
- CMOS time reading for game initialization and profiling
- Delay function for waiting on tick increments

## External Dependencies
- **DOS headers:** `<dos.h>`, `<mem.h>`, `<conio.h>` – DOS interrupt/I/O/memory macros
- **task_man.h:** `TS_ScheduleTask()`, `TS_Dispatch()`, `TS_Terminate()`, `TS_Shutdown()` – task scheduler
- **rt_in.h:** `LastScan`, `IN_ClearKeysDown()` – input state (defined elsewhere)
- **rt_def.h:** Constants (`VBLCOUNTER`, `MAXKEYBOARDSCAN`, `KEYQMAX`), types (`boolean`)
- **isr.h, _isr.h:** Interrupt vector numbers (`TIMERINT`, `KEYBOARDINT`)
- **keyb.h:** Keyboard constants (scroll_lock, num_lock, caps_lock, sc_* scan codes)
- **rt_main.h, rt_util.h, profile.h, develop.h:** Utilities and profiling (indirect)

# rott/isr.h
## File Purpose
Header file declaring interrupt service routines (ISRs) for keyboard input and timer management in a DOS/retro environment. Provides low-level hardware interrupt handling, keyboard state tracking, and frame timing via ticcount.

## Core Responsibilities
- Manage keyboard and timer interrupt service routine initialization/shutdown
- Provide circular queue for keyboard input buffering (KeyboardQueue)
- Track real-time keyboard state and detect key changes
- Maintain frame timing counter (ticcount, typically 70Hz)
- Control keyboard LEDs (num lock, caps lock, scroll lock)
- Provide utility functions for delays and timer configuration
- Expose lookup tables for ASCII-to-scancode conversion

## External Dependencies
- `keyb.h`: Scan code constant definitions (e.g., `sc_Return`, `sc_Escape`)
- Standard C runtime (implied interrupt/low-level hardware access)

# rott/keyb.h
## File Purpose
Defines a comprehensive mapping of keyboard scan codes to symbolic constant names. Provides a hardware abstraction layer for keyboard input by mapping PS2/DOS era scan codes (hex values) to descriptive identifiers (e.g., `sc_A`, `sc_Return`) used throughout the engine.

## Core Responsibilities
- Define symbolic constants for special keys (modifiers, arrows, function keys)
- Define symbolic constants for alphanumeric keys (letters 0-9)
- Map legacy PC keyboard scan codes to readable names
- Provide compile-time constants for keyboard input handling modules

## External Dependencies
- Standard C preprocessor directives only (`#ifndef`, `#define`, `#endif`)
- No external symbols; purely self-contained definitions

---

### Notes
- Uses legacy PC keyboard scan codes (IBM PS/2 style), typical for DOS-era games
- Maps both printable characters (letters, numbers, punctuation) and special keys (arrows, function keys, modifiers)
- Some redundancies: `sc_Enter` aliases `sc_Return` (0x1c), and `sc_Plus` aliases `sc_Equals` (0x0d)
- Guard `_keyb` prevents multiple inclusion
- Likely used by input capture and remapping systems; scan codes would be translated to game actions elsewhere

# rott/launch.h
## File Purpose
Header defining ProAudio Spectrum (PAS) audio driver interface for legacy DOS systems. Provides BIOS interrupt codes, mixer routing constants, and function declarations for driver detection and invocation using DOS far pointers and x86 register calling conventions.

## Core Responsibilities
- Define PAS driver communication codes (signature, interrupts, command opcodes)
- Define audio mixer input/output channel routing constants
- Declare function pointer table structure for driver audio operations
- Provide driver detection and low-level invocation functions

## External Dependencies
- Far pointer syntax and x86 register naming indicate 16-bit DOS/legacy code
- MV_* constants are ProAudio Spectrum BIOS interrupt codes (0xbc** range)
- No standard library includes

# rott/lookups.c
## File Purpose
Standalone utility that generates lookup tables for the ROTT renderer (pixel angles, sine, tangent, and gamma correction tables). These tables are precomputed at build time and written to a binary file for use by the game engine during runtime.

## Core Responsibilities
- Calculate pixel-to-angle mapping for raycasting (perspective correction)
- Generate sine and tangent lookup tables for trigonometric calculations
- Produce gamma correction curves for display brightness adjustment
- Write all four lookup tables to a binary output file in a fixed format

## External Dependencies
- **Includes:**
  - `rt_def.h`, `rt_util.h`, `rt_view.h` – local engine headers (defines constants like PANGLES, FINEANGLES, GAMMAENTRIES, GLOBAL1, PI).
  - `<math.h>` – Standard C math library (`sin()`, `tan()`, `atan()`, `pow()`).
  - `<dos.h>`, `<conio.h>`, `<io.h>` – DOS/legacy I/O APIs (`open()`, `write()`, `close()`).
  - `<fcntl.h>`, `<errno.h>`, `<sys/stat.h>` – POSIX file constants and error codes.
  - `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<stdarg.h>`, `<ctype.h>` – Standard C library.
  - `memcheck.h` – Memory debugging tool (no-op macros when disabled).
- **Defined elsewhere:**
  - `Error()`, `SafeOpenWrite()`, `SafeRead()`, `SafeWrite()` – Declared in `rt_util.h` header; implemented locally.

# rott/lumpy.h
## File Purpose
Public header defining typedef structures for graphics and font resources used throughout the ROTT engine. Contains data structure definitions for pictures, fonts, patches (sprites), and bitmap images—all core to the rendering and resource management system.

## Core Responsibilities
- Define in-memory representations of picture/sprite data (pic_t, lpic_t)
- Define font metadata and character rasterization structures (font_t, cfont_t)
- Define patch structures for sprite rendering with column-based offsets (patch_t, transpatch_t)
- Define lossless bitmap image format with embedded palette (lbm_t)
- Provide common type aliases across rendering and resource loading modules

## External Dependencies
- Standard C scalar types (byte, short, char, unsigned short)
- No explicit includes visible; assumes standard type definitions available in including translation units


# rott/mapsrot.h
## File Purpose
TED5 map header file defining the complete enumeration of playable levels in Rise of the Triad. Contains symbolic names for all campaign maps plus placeholder slots for future content. Serves as a centralized reference for map indices throughout the game engine.

## Core Responsibilities
- Define enum `mapnames` with all map identifiers
- Provide symbolic constants for map selection/loading
- Reserve slots (EMPTYMAP37–EMPTYMAP82) for map expansion
- Document map ordering and assignment (indices 0–83)

## External Dependencies
- No includes or external symbols visible in this file
- Assumed to be included by game logic modules that load/manage maps (likely `rott.h`, game state manager, or level loader)


# rott/memcheck.h
## File Purpose

Memory debugging library header file (MemCheck 3.0 Professional) that intercepts memory allocation/deallocation and string operations to detect buffer overflows, underflows, memory leaks, and invalid pointer usage. Provides cross-compiler DOS support with extensive configuration and callback mechanisms.

## Core Responsibilities

- Define memory tracking data structures (MEMREC) and error codes (MCE_*)
- Detect compiler type and memory model, establish abstraction layer for 16/32-bit code
- Intercept standard C library functions (malloc, free, strcpy, etc.) via macro redefinition
- Declare MemCheck API functions for initialization, checking, and reporting
- Provide callback registration for custom error handling, tracking, and validation
- Support both C and C++ (with overloaded new/delete operators)
- Define link-time configuration macros (MC_SET_*) for compile-time settings

## External Dependencies

- **Compiler detection**: Preprocessor symbols from MSC, Borland, Watcom, Intel, ANSI C compilers
- **Standard headers**: stdio.h (NULL, FILE), stdarg.h (va_list), malloc.h / alloc.h (compiler-specific), string.h
- **Assumed OS**: 16-bit or 32-bit DOS; real mode or protected mode (DPMI, Phar Lap)
- **Defined elsewhere**: RTL replacements (malloc_mc, free_mc, etc.) in MemCheck libraries; exception handlers; stock callbacks (erf_default, trackf_all, etc.)
- **Linker-time**: MC_SET_* macro instantiations in user code link with MemCheck library globals

---

**Notes**: This is a highly sophisticated legacy debugging tool with cross-compiler abstraction, compiler-specific intrinsic disabling pragmas, and careful macro design to preserve source locations through multi-layer interception. The dual-mode design (MEMCHECK vs. NOMEMCHECK) allows zero-overhead production builds. Extensive support for 16-bit segmented memory models (far/near pointers) and multitasking critical sections reflects DOS-era constraints.

# rott/modexlib.c
## File Purpose
Mode-X VGA graphics library providing low-level video initialization, buffer management, and page-flipping for DOS 320x200 256-color mode. Handles CPU-to-VRAM transfers, planar memory layout configuration, and synchronization with display hardware.

## Core Responsibilities
- Switch CPU between graphics and text modes via BIOS interrupt 0x10
- Configure VGA hardware registers for non-chained planar mode (Mode-X)
- Manage three video memory pages for double/triple buffering
- Perform synchronization with vertical blank (VBL) timing
- Copy framebuffer data between planar VRAM and linear system memory
- Clear and fill video memory with specified colors
- Execute display page flips with address register updates

## External Dependencies
- **Headers**: `<dos.h>` (BIOS interrupt dispatch via `int386`, `union REGS`); `<string.h>` (`memcpy`, `memset`); `<stdio.h>`, `<stdlib.h>`, `<malloc.h>` (I/O, memory, standard utilities).
- **Local headers**: `modexlib.h` (VGA register constants, macro definitions for `VGAREADMAP`, `VGAWRITEMAP`, `VGAMAPMASK`); `memcheck.h` (memory debugging wrapper—inert in this file).
- **External symbols**: `outp()`, `outpw()`, `inp()` (port I/O); `int386()` (interrupt dispatch); `memcpy()`, `memset()` (memory operations).

# rott/modexlib.h
## File Purpose
Public header for the ModeX video library, providing VGA register constants, framebuffer state variables, and functions for graphics/text mode switching, buffer management, page flipping, and direct VGA hardware control. ModeX is a specialized VGA planar mode (320×200, 256 colors) commonly used in DOS games of this era.

## Core Responsibilities
- VGA hardware register port definitions (Sequencer, CRTC, Graphics Controller, Attribute, Palette)
- Screen geometry and memory layout constants
- Global video state variables (framebuffer offsets, page pointers, line width, mode flags)
- Mode switching (graphics ↔ text)
- Framebuffer operations (clear, copy, page flipping)
- Vertical blank synchronization
- Direct VGA plane selection via inline assembly wrappers

## External Dependencies
- `rt_def.h`: Base types (byte, boolean, int), game constants (MAXSCREENHEIGHT, MAXSCREENWIDTH, etc.)
- VGA hardware: Direct I/O ports in range 0x3C0–0x3DA (Sequencer, CRTC, Graphics Controller, Attribute, Palette, Status)
- Implementations: All function bodies reside in modexlib.c (not in this header)

# rott/mouse.h
## File Purpose
This is a header file for mouse input handling in the ROTT (Rise of the Triad) game engine. The provided content contains only the GPL license header; the actual interface definitions are not included in the excerpt.

## Core Responsibilities
- Not inferable from this file (license header only).

## External Dependencies
- No dependencies visible in provided content.

---

**Note:** The provided file content contains only the GPL v2 license header (copyright 1994-1995 Apogee Software). To analyze the actual architectural role of `mouse.h`, please provide the complete file including its type definitions, function declarations, and data structure definitions.

# rott/music.h
## File Purpose
Public API header for the MUSIC subsystem. Declares functions for initializing MIDI/music devices, playback control, volume management, song positioning, and real-time MIDI channel manipulation in the ROTT game engine.

## Core Responsibilities
- Initialize and shutdown music/MIDI hardware
- Control music playback state (play, pause, continue, stop)
- Manage global and per-channel volume levels
- Support song position seeking (by ticks, milliseconds, or measure/beat/tick)
- Provide fade-out effects and loop control
- Support MIDI channel rerouting to custom handlers
- Register timbre/instrument banks
- Expose error codes and context switching

## External Dependencies
- `#include "sndcards.h"` — Provides `soundcardnames` enum for sound card types
- MIDI/FM synthesis hardware interface (implementation in MUSIC.C, not visible here)
- DOS/protected-mode conventions (cdecl callbacks, hardware I/O addresses)

# rott/myprint.h
## File Purpose
Text rendering and output module for the game engine. Provides a color-aware text drawing interface for displaying characters, strings, formatted output, and framed text boxes on-screen.

## Core Responsibilities
- Define standard 16-color palette (BLACK through WHITE)
- Provide low-level character/string output at screen coordinates
- Support formatted printf-style string printing
- Draw text boxes and frames with borders
- Manage text cursor positioning

## External Dependencies
- Underlying screen buffer or console interface (implementation not visible)
- Assumes 16-color DOS-style text mode

# rott/profile.h
## File Purpose
A minimalist profiling/performance measurement configuration header. Defines preprocessor constants that control profiling behavior throughout the engine.

## Core Responsibilities
- Enable/disable profiling instrumentation via `PROFILE` macro
- Define profiling tick frequency or sampling interval via `PROFILETICS`

## External Dependencies
- Standard C preprocessor directives only; no external includes
- Licensed under GNU GPL v2

**Notes:**  
- Extremely lightweight—only two constants, both currently inactive/minimal
- `PROFILE=0` suggests profiling is disabled by default (likely omitted from release builds)
- `PROFILETICS=2` purpose not inferable without seeing usage in instrumented code; possibly a tick multiplier, sample rate divisor, or interval threshold

# rott/r_scale.asm
## File Purpose
Low-level x86 assembly implementation of scaled texture column rendering. Draws vertical slices of textures with interpolation/filtering, operating on fixed-point texture coordinates. Part of the software raycasting/scanline renderer pipeline.

## Core Responsibilities
- Compute texture coordinate interpolation using fixed-point arithmetic
- Fetch and write texture pixels with vertical scaling/filtering
- Process columns in pairs for cache and loop efficiency
- Handle variable column heights and vertical positioning within screen buffer
- Self-modify scale increment instructions for runtime configuration

## External Dependencies
- **Includes/Directives:** None (pure assembly).
- **External symbols:** `_dc_yl`, `_dc_yh`, `_dc_ycenter`, `_dc_iscale`, `_dc_texturemid`, `_ylookup`, `_dc_source` (defined elsewhere; likely C/C++ globals set by rendering state machine).
- **Assumptions:** Caller sets `edi` to screen buffer base; linear video memory layout.

# rott/rand.asm
## File Purpose
Implements table-based random number generation via lookup table and index cycling. Provides initialization with optional time-based seeding and fast O(1) random value retrieval (0–255) suitable for real-time gameplay.

## Core Responsibilities
- Maintain a 256-entry precomputed random value lookup table
- Initialize the RNG with deterministic (index=0) or time-seeded (via DOS INT 21h) start
- Provide fast random number generation by table lookup and index cycling
- Track and wrap the current table index (0–255)

## External Dependencies
- INT 21h (DOS/BIOS system time interrupt) via `US_InitRndT_`
- x86 32-bit processor mode (`.386p` directive)
- Flat memory model (`.model flat`)
- No C runtime or external symbol dependencies

# rott/rottnet.h
## File Purpose
Defines the networking protocol and communication interface between the ROTT game engine and an external network driver. Establishes shared data structures, constants, and function declarations for multiplayer session management across up to 14 networked nodes supporting both modem and network game modes.

## Core Responsibilities
- Define the `rottcom_t` structure for game-to-driver command/data exchange
- Establish networking constraints (max players, packet sizes, buffer limits)
- Declare driver-level networking functions (ISR, launch, shutdown, vector management)
- Provide conditional compilation for shareware vs. full product and Watcom-specific packing
- Define I/O port constants and helper macros for palette management
- Distinguish between server and client roles in multiplayer sessions

## External Dependencies
- **Conditional includes**: `develop.h` (Watcom builds, debug config) or `global.h` (shareware/retail, shared types).
- **I/O functions**: `outp()`, `inp()` (legacy DOS/x86 port I/O macros, defined in global.h or compiler runtime).
- **Types**: `boolean`, `short`, `char`, `long` (defined in global.h).
- **Watcom-specific**: `#pragma pack` for struct alignment; conditional `rottcom` pointer vs. direct reference depending on compiler.
- **External symbols used but not defined here**: `rottcom` (shared memory region or pointer), `pause` flag, interrupt handler setup, process spawning.

# rott/rottser.h
## File Purpose
Header file defining serial port configuration data structure. Used to store and pass serial communication parameters (IRQ, UART base address, and baud rate) throughout the game engine.

## Core Responsibilities
- Define the serial port configuration structure
- Provide a standard type for serial device setup
- Encapsulate hardware-level serial communication parameters

## External Dependencies
- Standard C library headers (implicitly included by files that use this header)
- No engine-specific dependencies visible

---

**Notes:**
- The three fields (`irq`, `uart`, `baud`) suggest DOS-era serial port configuration where IRQ levels and UART I/O addresses were manually specified
- `long` type for all fields suggests 32-bit values on the target platform
- This is part of a serial communication subsystem, likely for network multiplayer or modem connectivity

# rott/rt_actor.c
## File Purpose
Implements the actor system for managing all dynamic game entities including enemies, projectiles, environmental hazards, and interactive objects. Handles spawning, physics, collision, damage, state management, AI behavior, and save/load persistence for the game world.

## Core Responsibilities
- **Actor lifecycle management**: Allocation, initialization, cleanup, and free list maintenance for actor objects
- **Physics simulation**: Gravity, momentum, friction, collision detection and response, Z-axis movement
- **State machine execution**: State transitions, think function dispatch, animation frame advancement per actor class
- **Collision system**: Multi-stage collision checks (actors, walls, doors, static objects, masked walls) with response generation
- **Damage and death**: Player/enemy damage handling, fatality sequences, gib spawning, death state transitions
- **Projectile system**: Missile spawning, movement, collision detection, area-of-effect explosions, hitscan weapon traces
- **AI behavior**: Pathfinding, chase logic, attack decision-making, special boss behaviors (Darian, Heinrich, Oscuro)
- **Spatial organization**: Per-area actor linked lists, tile-based actor map, active/inactive actor tracking
- **Save/restore state**: Actor serialization/deserialization for game persistence

## External Dependencies
- **rt_def.h**: Core type definitions (`objtype`, `classtype`, `exit_t`, flag constants)
- **rt_sound.h**: Sound playback (`SD_PlaySoundRTP`, `SD_StopSound`, `SD_SoundActive`)
- **rt_door.h**: Door/elevator structures, collision types (`doorobj_t`, `maskedwallobj_t`)
- **rt_ted.h**: Level data access (`MAPSPOT`, `AREANUMBER`, platform checks)
- **rt_draw.h**: Rendering support (`SetVisiblePosition`, `SetFinePosition`, lighting)
- **states.h**: State table definitions (`statetype`, state pointers like `&s_chase1`)
- **sprites.h**: Sprite/animation management (`TurnActorIntoSprite`, `PreCacheActor`)
- **gmove.h**: Movement utilities (`FindDistance`, `atan2_appx`, angle/direction tables)
- **rt_game.h**: Game state (`gamestate`, `MISCVARS`, `PLAYER[0]`)
- **Defined elsewhere**: `FIRSTSTAT`, `LASTSTAT` (sprite list); `doorobjlist[]`, `maskobjlist[]`, `pwallobjlist[]` (object pools); `tilemap[][]` (collision grid); `costable/sintable` (trig lookup)

# rott/rt_actor.h
## File Purpose
Public header for the actor (game entity) system in ROTT. Defines the core `objtype` structure representing any dynamic object (enemies, player, projectiles, hazards), declares spawn/despawn/movement/collision functions, and provides macros for actor state management and position queries.

## Core Responsibilities
- Define actor object structure (`objtype`) and classify entities by type (`classtype` enum)
- Manage actor linked lists (FIRSTACTOR/LASTACTOR, area actors, active/inactive actors)
- Spawn and despawn actors (enemies, hazards, projectiles, particles)
- Provide position manipulation macros (tile-based and fine-grained)
- Handle collision detection and response (ActorTryMove, MissileTryMove, Collision)
- Damage calculation and actor death (DamageThing, KillActor, MissileHit)
- State machine integration (NewState, DoActor) for AI and behavior
- Pathfinding and chase logic (SelectPathDir, SelectChaseDir, SightPlayer)

## External Dependencies
- **Include:** `states.h` — State machine structures (statetype) and state instance externs (s_lowgrdstand, s_chase1, etc.)
- **Defined elsewhere (used here):**
  - `statetype` struct and state instances
  - `thingtype`, `dirtype` enums (basic types)
  - BATTLE_CheckGameStatus, sound system functions
  - TILESHIFT, TILEGLOBAL, MAPSIZE constants (map module)
  - Memory allocator (GetNewActor)
  - Trigonometric/angle utilities (angletodir[], AngleBetween)

# rott/rt_battl.c
## File Purpose
Implements battle/multiplayer game mode support for Rise of the Triad, including mode initialization, round management, kill tracking, score calculation, and game-state transitions based on battle events. Manages team assignments, point goals, and special game rules for deathmatch variants.

## Core Responsibilities
- Initialize battle system with selected mode (Normal, Tag, Hunter, Collector, etc.) and player configuration
- Track and update player/team points, kills, and rankings across rounds
- Handle game state transitions triggered by battle events (kills, item collection, time limits)
- Validate battle mode rules and enforce friendly-fire/spawn settings
- Sort player rankings and synchronize score displays
- Calculate points awarded based on kill type and battle mode
- Manage round lifecycle (start, refresh timer, end conditions)

## External Dependencies
- **Global state:** `gamestate` (BattleOptions, ShowScores, SpawnHealth, SpawnWeapons, SpawnDangers, SpawnCollectItems, Product, SpecialsTimes); `PLAYERSTATE[]` (player colors/uniforms); `consoleplayer` (current player index); `GRAVITY`
- **Functions defined elsewhere:**
  - `GameRandomNumber()` (rt_rand.c) – random number generation
  - `SD_Play()` (rt_sound.c) – play sound effect
  - `DrawKills()` (rt_view.c) – render score display
  - `SHOW_TOP_STATUS_BAR()`, `SHOW_KILLS()` (rt_view.c) – UI visibility checks
  - `AddMessage()` (rt_msg.c) – game message queue
  - `RespawnEluder()`, `SpawnCollector()` (rt_actor.c) – NPC spawn/respawn
  - `Error()`, `SoftError()` (debugging macros, conditional on BATTLECHECK/BATTLEINFO flags)
  - `MINUTES_TO_GAMECOUNT()` (macro in rt_battl.h) – time conversion
- **Notable includes:** rt_def.h (constants, types), rottnet.h (networking), isr.h (timer constants), rt_battl.h (public interface), rt_actor.h, rt_rand.h, rt_playr.h, rt_game.h, rt_sound.h, rt_com.h, rt_msg.h, rt_view.h, rt_util.h, rt_main.h, memcheck.h

# rott/rt_battl.h
## File Purpose
Public header for the battle system (multiplayer modes) in Rise of the Triad. Defines battle mode types, configuration options, event codes, and declarations for initializing and managing multiplayer/deathmatch gameplay.

## Core Responsibilities
- Define battle modes (Normal, Collector, Scavenger, Hunter, Tag, Eluder, Deluder, Capture The Triad, etc.)
- Declare battle event types and status return codes
- Define battle configuration options (speed, ammo, hit points, light levels, kill limits, damage)
- Expose battle system state variables (kill tracking, player points, team assignments)
- Declare core battle lifecycle functions (init, shutdown, event handling)

## External Dependencies
- Assumes `MAXPLAYERS` constant defined elsewhere
- Assumes `VBLCOUNTER` defined (time reference)
- References RT_MENU.C for `BATTLE_Options` array initialization

# rott/rt_build.c
## File Purpose
Implements a 3D menu rendering system that projects textured planes in perspective and renders 2D UI elements (text, sprites, primitives) onto a double-buffered menu surface. Provides the backbone for animated menu screens with rotating 3D backgrounds.

## Core Responsibilities
- Menu buffer lifecycle (initialization, clearing, shutdown)
- 3D-to-2D perspective projection for planar geometry
- Depth-sorted plane rendering with affine texture mapping
- Double-buffering infrastructure for smooth menu transitions
- Drawing primitives (boxes, lines, pixels)
- Text rendering with multiple shading and intensity modes
- Sprite and picture composition onto the menu buffer
- Menu animation and flip transitions

## External Dependencies
- **Includes:** `RT_DEF.H` (constants, types), `rt_draw.h` (visobj_t, drawing state), `watcom.h` (fixed-point math), `lumpy.h` (WAD structures), `w_wad.h` (resource loading), `rt_util.h`, `rt_vid.h`, `rt_sound.h`, `modexlib.h` (video/sound)
- **Extern symbols (defined elsewhere):**
  - `costable[]`, `sintable[]`: precomputed trig lookup tables
  - `W_CacheLumpNum()`, `W_CacheLumpName()`, `W_GetNumForName()`: WAD resource management
  - `posts[]`: per-column screen state (rt_draw.c)
  - `colormap`, `playermaps[]`: color lookup tables
  - `Keyboard[]`: input state array
  - `Menuflipspeed`: configuration variable
  - `CurrentFont`, `IFont`: active font pointers
  - `tics`, `viewwidth`, `viewheight`, `centery`, etc.: rendering state from rt_draw.c
  - `ylookup[]`: scanline offset table
  - `bufferofs`, `screenofs`: video memory pointers
  - `PrintX`, `PrintY`: text cursor state
  - `shadingtable`: active shading palette (local and extern)

# rott/rt_build.h
## File Purpose
Public header declaring the menu buffer management interface for the ROTT engine. Provides functions to initialize, render, and draw UI elements (shapes, text, pictures) to an off-screen menu buffer that can be positioned and displayed with transparency/intensity effects.

## Core Responsibilities
- Menu buffer lifecycle management (setup, shutdown, clear)
- Menu buffer positioning and animation (angle, distance, refresh timing)
- Drawing primitives to the menu buffer (shapes, pictures, text in multiple styles)
- Text rendering variants (proportional, transparent, colored, shaded)
- Menu UI element erasure and region management

## External Dependencies
- `byte` type (likely defined in a common types header; maps to `unsigned char`)
- `boolean` type (likely `typedef` for `int` or similar)
- Implementation in `rt_build.c`

# rott/rt_cfg.c
## File Purpose
Configuration manager for Rise of the Triad that loads/saves user settings from disk files (CONFIG.ROT, SOUND.ROT, BATTLE.ROT). Handles parsing text configuration files, managing sound hardware settings, input device mappings, game preferences, battle mode rules, and password encryption.

## Core Responsibilities
- Parse and write configuration script files (SOUND.ROT, CONFIG.ROT, BATTLE.ROT)
- Manage sound hardware settings (Sound Blaster, MIDI, sample rates, volumes)
- Manage input device configuration (mouse, keyboard, joystick calibration)
- Persist player preferences (detail levels, control mappings, visual effects)
- Configure battle mode rules and special powerup timings
- Encrypt/decrypt game passwords and violence level settings
- Locate alternate content via SETUP.ROT (remote sounds, game/battle levels)
- Verify vendor documentation integrity via CRC checks
- Provide sensible defaults when config files are missing

## External Dependencies
- **scriplib.h:** `LoadScriptFile()`, `GetToken()`, `GetTokenEOL()`, `TokenAvailable()`, `scriptbuffer`
- **w_wad.h:** `W_GetNumForName()`, `W_CacheLumpNum()`, `W_LumpLength()` (WAD file access)
- **z_zone.h:** `Z_Free()` (memory management)
- **rt_crc.h:** `CalculateCRC()` (checksum computation)
- **rt_sound.h:** Sound system (device type constants, `FX_GetBlasterSettings()`)
- **rt_in.h:** `IN_SetupJoy()` (joystick calibration)
- **rt_util.h:** File utilities (`SafeOpenRead()`, `SafeOpenWrite()`, `SafeRead()`, `SafeWrite()`, `SafeWriteString()`, `SafeFree()`, `LoadFile()`, `SaveFile()`, `GetPathFromEnvironment()`)
- **rt_playr.h:** Defines `MAXCODENAMELENGTH`, weapon types
- **rt_game.h:** Game state struct (`gamestate`, `BATTLE_Options[]`, `BATTLE_ShowKillCount`, `BattleSpecialsTimes`)
- **rt_main.h:** Control state (`buttonscan[]`, `buttonmouse[]`, `buttonjoy[]`, `joyxmin`, etc.)
- **rt_battl.h:** Battle mode enums and constants
- **rt_msg.h:** `MessagesEnabled` flag
- **rt_view.h:** `gammaindex`, `fulllight`
- **develop.h:** Development constants
- **memcheck.h:** Memory checking (MED)
- **POSIX/BIOS C libraries:** `<io.h>`, `<fcntl.h>`, `<conio.h>`, `<process.h>` (DOS-era file I/O, terminal, process control)

# rott/rt_cfg.h
## File Purpose
Public configuration header for the runtime settings management module. Declares all global configuration variables for audio, input devices, graphics, and gameplay settings, along with functions to persist these settings to disk. Also defines structures for alternate resource loading (sounds, graphics, levels) and combat macros.

## Core Responsibilities
- Declare and expose global configuration variables (audio modes, volumes, input device settings, graphics quality, player preferences)
- Define data structures for alternate resource information and macro commands
- Provide public interface for reading/writing configuration files, scores, and battle configuration
- Handle password/difficulty/character selection persistence and conversion
- Support sound file and vendor management

## External Dependencies
- Standard C types: `int`, `char`, `byte`, `boolean` (boolean defined elsewhere, likely in common header)
- File I/O: assumed in implementation
- Game-specific paths and resource management

# rott/rt_com.c
## File Purpose
Network communication and time synchronization module for Rise of the Triad's multiplayer system. Handles packet I/O through DOS interrupt calls to a real-mode network driver, and implements a 5-phase clock synchronization protocol between master (server) and slave (client) players.

## Core Responsibilities
- Initialize network interface and locate shared real-mode COM structure
- Read and write game packets with CRC integrity checks
- Manage 5-phase time synchronization handshake between master and slave
- Calculate and store round-trip transit times for latency compensation
- Handle server/client role transitions in packet addressing
- Validate incoming sync packets and coordinate phase progression
- Broadcast start commands and wait for player readiness during game init

## External Dependencies
- **DOS/System headers:** `<dos.h>`, `<conio.h>`, `<process.h>`, `<bios.h>` (real-mode interrupt interface)
- **Game headers:** 
  - `rt_def.h` — global constants and types
  - `rt_util.h` — utilities (malloc, error, command-line parsing)
  - `rt_crc.h` — `CalculateCRC()`
  - `isr.h` — `ISR_SetTime()`
  - `rt_main.h` — `ticcount` global
  - `rt_playr.h` — `PlayerInGame()`, `numplayers`
  - `rottnet.h` — `rottcom_t`, network constants
  - `rt_msg.h` — `AddMessage()`, `MSG_SYSTEM`
  - `rt_draw.h` — `ThreeDRefresh()`
- **External symbols (defined elsewhere):**
  - `int386()` — DOS real-mode interrupt dispatcher
  - `ticcount` — global game time counter (in tics)
  - `networkgame`, `IsServer`, `standalone` — global game mode flags
  - `quiet` — debug flag
  - `server` — server player index constant
  - `_argv[]`, `_argc` — command-line arguments

# rott/rt_com.h
## File Purpose
Public interface for ROTT networking and packet communication. Declares initialization routines, packet I/O functions, and synchronization utilities for multi-player game sessions.

## Core Responsibilities
- Initialize the network subsystem (`InitROTTNET`)
- Read and write game network packets
- Manage packet buffers and transit timing
- Expose global networking state (player ID, sync clock)

## External Dependencies
- `rottnet.h` — Defines `rottcom_t`, platform macros (`__WATCOMC__`), and max buffer sizes
- Implies a driver/platform layer (DOS/network driver for modem or LAN games)

# rott/rt_crc.c
## File Purpose
CRC-16 checksum calculation library for data integrity verification. Provides functions to compute Cyclic Redundancy Check values using a precomputed lookup table for efficient streaming and batch computation.

## Core Responsibilities
- Compute CRC-16 checksums for data buffers
- Support incremental (single-byte) CRC updates via lookup table
- Provide high-performance checksum calculation without dynamic allocation

## External Dependencies
- **Standard includes:** stdio.h, stdlib.h, string.h
- **Local includes:** rt_crc.h (function declarations), memcheck.h (debugging header, not functionally used)
- **Defined elsewhere:** `byte`, `word` types (likely in rt_def.h per rt_crc.h include chain)

# rott/rt_crc.h
## File Purpose
Header file declaring the CRC (Cyclic Redundancy Check) calculation interface. Provides function signatures for incremental CRC updates and bulk buffer CRC computation used for data integrity verification.

## Core Responsibilities
- Declare incremental CRC update function
- Declare block CRC calculation function for byte buffers
- Export CRC utility interface to other engine modules

## External Dependencies
- Includes: `rt_def.h` (provides `byte`, `word` type definitions)
- Implementation defined elsewhere (rt_crc.c presumed)

# rott/rt_debug.c
## File Purpose

Implements cheat code detection and execution for the Rise of the Triad game engine. Processes keyboard input to recognize cheat sequences and triggers corresponding debug/cheat effects (godmode, weapons, level warping, demo recording, etc.). Also provides frame-by-frame development debug keys when DEVELOPMENT mode is enabled.

## Core Responsibilities

- Detect and validate cheat code sequences from keyboard input
- Execute cheat effects (god mode, invulnerability, weapons, armor, powerups)
- Handle level warping and game state modifications
- Support demo recording, playback, and jukebox access
- Manage game state toggles (light diminishing, fog, HUD, missile cam, floor/ceiling rendering)
- Provide frame-by-frame debug controls for development builds
- Spawn bonus items and manage player health/armor/keys via cheat spawning

## External Dependencies

**Notable Includes / Imports:**
- `rt_def.h` – core engine definitions (flags, types, constants)
- `rt_game.h` – game state, player management, scoring, rendering  
- `rt_menu.h` – menu system (control panel, level selection, jukebox)
- `rt_in.h` – input management
- `rt_playr.h` – player object structure
- `rt_build.h` – level/build data
- `rt_draw.h`, `rt_vid.h` – rendering
- `rt_view.h` – viewport/camera
- `rt_sound.h`, `rt_msg.h`, `rt_net.h`, `rt_stat.h`, `rt_map.h` – sound, messaging, networking, entity/map state
- `isr.h` – keyboard/interrupt input (Keyboard array, IN_UpdateKeyboard, etc.)
- `z_zone.h` – memory management
- `develop.h` – development utilities
- `memcheck.h` – memory debugging

**Defined Elsewhere, Used Here:**
- Global arrays/vars: `Keyboard[]`, `LetterQueue[]`, `PLAYER[]`, `PLAYERSTATE[]`, `LASTSTAT`, `MAPSPOT()`
- Functions: `SpawnStatic()`, `MakeStatActive()`, `AddMessage()`, `DamageThing()`, `HealPlayer()`, `DrawKeys()`, `GivePoints()`, `CheatMap()`, `CP_LevelSelectionMenu()`, `MU_JukeBoxMenu()`, `MU_StartSong()`, `MU_RestoreSongPosition()`, `SaveDemo()`, `LoadDemo()`, `DemoExists()`, `RotationFun()`, `ThreeDRefresh()`, `DoSprites()`, `CalcTics()`, etc.
- Globals: `player`, `gamestate`, `locplayerstate`, `godmode`, `missilecam`, `HUD`, `fandc`, `demorecord`, `demoplayback`, `ludicrousgibs`, `DebugOk` (extern), `CurrentFont`, `smallfont`

# rott/rt_debug.h
## File Purpose
Header file declaring debug and cheat code subsystem entry points. Provides an interface for managing cheat codes, processing debug input, and controlling demo playback within the game engine.

## Core Responsibilities
- Declare cheat code initialization/reset function
- Declare debug key input handler
- Declare debug status check function
- Declare demo termination function

## External Dependencies
- No explicit includes shown (standard C header convention)
- Implementation in `rt_debug.c` (inferred from file comment)
- Assumes external input and demo systems exist

# rott/rt_def.h
## File Purpose
Global constant and type definitions for the ROTT engine. Defines fundamental mathematical constants (angles, distances), screen/view dimensions, game entity types, input controls, weapon types, and flag bits for actors, sprites, and game state. Central foundation included by most engine modules.

## Core Responsibilities
- Define engine-wide constants: view dimensions (320×200), angle system (2048 angles), tile/pixel scaling units
- Define fundamental data types used throughout the codebase (`byte`, `word`, `fixed`, `boolean`)
- Enumerate game entity types (actor, sprite, wall, door)
- Enumerate input controls (attack, strafe, look, weapons, movement)
- Enumerate weapon types (pistol, MP40, bazooka, special weapons)
- Define flag bits for entity attributes (shootable, active, dying, etc.)
- Define map constants (tile size, map dimensions, tile IDs)
- Provide macro helpers for register access, map indexing, and area calculations

## External Dependencies
- **`<stdio.h>`** – Standard I/O (likely used elsewhere, not directly in this file)
- **`"develop.h"`** – Development configuration flags (SHAREWARE, SUPERROTT, SITELICENSE; controls conditional weapon/game-mode compilation)

---

### Notes
- **Fixed-point math**: `SFRACBITS`, `SFRACUNIT`, `FRACUNIT` indicate 16-bit fractional scaling for sub-tile precision
- **Angle system**: 2048 fine angles divided into quads; `ANG90`, `ANG180`, etc. provide cardinal angle constants for both fine and coarse angle systems
- **Weapon branching**: Shareware build has 9 weapons; full version adds 4 more (split, kes, bat, dog)
- **Macro helpers**: `MAPSPOT()`, `AREANUMBER()` abstract map access; `AX/BX/CX/DX/SI/DI` macros suggest DOS-era x86 register structs
- **Device support**: SWIFT constants suggest input device abstraction (Cyberman support)

# rott/rt_dmand.c
## File Purpose
Manages demand-fed streaming audio for both playback (remote voice transmission) and recording (voice capture for remote transmission) in networked gameplay. Implements circular buffer management to handle audio chunks efficiently, with callback-based integration to the FX (sound effects) subsystem.

## Core Responsibilities
- Initialize and manage playback/recording buffers for network audio transmission
- Handle circular buffer pointer management for streaming audio chunks
- Provide callback integration points for the audio subsystem (`SD_UpdatePlaybackSound`, `SD_UpdateRecordingSound`)
- Retrieve recorded audio data for network transmission via `SD_GetSoundData`
- Synchronize playback/recording state with network and audio device state
- Manage semaphore flags for cross-system recording activation

## External Dependencies
- **FX subsystem**: `FX_StartDemandFeedPlayback`, `FX_StopSound`, `FX_StartRecording`, `FX_StopRecord` (defined elsewhere)
- **Memory**: `SafeMalloc`, `SafeFree` (rt_util.h)
- **Global state**: `SD_Started` (rt_sound.h), `remoteridicule` (rt_net.h)
- **Debug**: `whereami` extern (develop.h)
- **Headers**: Includes rt_def.h (types), rt_util.h, rt_sound.h, rt_net.h, _rt_dman.h, fx_man.h, develop.h, memcheck.h

# rott/rt_dmand.h
## File Purpose
Header file for sound demand management, providing interfaces to receive incoming sound data in chunks and manage sound recording. Part of the game engine's audio streaming subsystem (SD_ prefix indicates sound driver functions).

## Core Responsibilities
- Establish and teardown streaming audio input pipelines
- Buffer and retrieve incoming sound data in discrete chunks
- Query audio data availability and stream state
- Manage sound recording state (active/inactive)
- Coordinate recording start/stop and data flow

## External Dependencies
- Primitive types: `byte`, `word`, `boolean` (defined elsewhere; typical C89 typedefs for DOS/early platform compatibility)
- No external includes visible in this header

# rott/rt_door.c
## File Purpose
Manages all door, elevator, push-wall, masked-wall, and touch-plate (trigger) systems for the game. Handles spatial state (open/closed, area connectivity), player interaction, and save/load persistence of these interactive elements.

## Core Responsibilities
- **Door system**: Spawn, open, close, lock, check collisions; manage multi-door linkage; update `areaconnect` matrix when doors transition between open/closed states
- **Elevator system**: Manage two-location fast-travel with locked doors; handle player requests; teleport actors/statics between locations; optionally play elevator music
- **Push-wall system**: Spawn and update pushable/moving walls; track momentum and state; connect areas when walls move; handle collision resolution
- **Masked-wall system**: Manage transparent/shootable walls with animated frame sequences; handle break animations; maintain active/inactive linked lists
- **Touch-plate/trigger system**: Link actions (door open/close, push walls, lights, objects) to tile triggers; manage action queues and state callbacks
- **Area connectivity**: Maintain and update `areaconnect[][]` bidirectional connectivity matrix; compute `areabyplayer[]` via recursive traversal; determine visibility and sound propagation

## External Dependencies

**Notable includes / imports:**
- `rt_def.h` – core constants (MAPSIZE, NUMAREAS, direction enums, flags)
- `rt_sound.h` – sound system (SD_Play, SD_PlaySoundRTP, SD_PanRTP, MU_StartSong, MU_RestoreSongPosition)
- `rt_actor.h` – actor types and macros (M_ISACTOR, objtype, classtype, actor lists)
- `rt_stat.h` – static object types (statobj_t, stat_t enum)
- `rt_ted.h` – tile editor data structures
- `z_zone.h` – memory allocation (Z_LevelMalloc, Z_Free)
- `w_wad.h` – WAD file access (W_GetNumForName, W_CacheLumpNum)
- `rt_draw.h` – rendering hints
- `rt_main.h` – global game state (gamestate, tics, loadedgame, insetupgame)
- `rt_playr.h` – player object type (player, PLAYER[])
- `rt_util.h` – utilities
- `rt_menu.h` – HUD messages (AddMessage)
- `rt_msg.h` – message types
- `rt_game.h` – game state
- `rt_vid.h` – video I/O
- `rt_net.h` – network/multiplayer
- `isr.h` – interrupts
- `develop.h` – debug/development flags
- `rt_rand.h` – random number generation
- `engine.h` – 3D engine
- `stdlib.h`, `string.h` – C standard library (memcpy, memset)
- `memcheck.h` – memory debugging

**Defined elsewhere (not in this file):**
- `actorat[][]`, `sprites[][]`, `tilemap[][]`, `mapplanes[][]` – global map data
- `firstactive`, `firstactivestat`, `FIRSTSTAT`, `LASTSTAT` – object linked lists
- `PLAYER[]`, `numplayers` – player array
- `gamestate`, `MISCVARS` – global game state structs
- `spotvis[][]`, `mapseen[][]` – visibility tracking
- `walls[]`, `GetWallIndex()` – wall texture database
- `objlist[]` – actor pointer array (for save/load)
- `Clocks[]`, `numclocks` – clock/timer system (referenced but not defined here)
- `demoplayback`, `demorecord` – demo flags
- `loadedgame`, `insetupgame` – state flags
- `SHAKETICS` – screen shake counter
- `firstactivestat`, `lastactivestat` – stat active list
- `BATTLE_CheckGameStatus()`, `BATTLEMODE` – battle mode system
- Sound functions: `SD_StopSound()`, `MU_StoreSongPosition()`, `MusicStarted()`, `GameRandomNumber()`
- Other: `Error()`, `SoftError()`, `Debug()` – diagnostics
- Rendering: `VL_DecompressLBM()`, `VW_UpdateScreen()`, `I_Delay()`

# rott/rt_door.h
## File Purpose
Header file for the ROTT game engine's door, wall, and elevator system. Defines data structures and function prototypes for interactive map objects including sliding doors, pushwalls, masked walls, elevators, and trigger plates.

## Core Responsibilities
- Define data structures for doors, pushwalls, elevators, masked walls, and touch-activated triggers
- Declare initialization functions for doors, elevators, and area connectivity
- Declare runtime update functions (door/wall/elevator movement, animation, state machine)
- Declare save/load functions for persistent state serialization
- Declare collision/area helper functions and query macros
- Define wall type flags and state enumerations

## External Dependencies

- **Notable includes / imports:** None explicit in header; likely includes C standard library, game types (`thingtype`), and map headers
- **Defined elsewhere:**
  - `thingtype`: Likely enum or typedef for game object types
  - `tilemap[x][y]`: Global 2D array representing the map grid (bits encode door, wall, door-type info)
  - `NUMAREAS`, `MAPSIZE`: Preprocessor constants defining map dimensions
  - Sound system (`int soundhandle`): Likely external audio API
  - Texture system (`word texture`, `word alttexture`, `int sidepic`): Likely external rendering API
  - `tiling`, `area`, and collision geometry: Likely defined in map/collision modules

# rott/rt_dr_a.asm
## File Purpose
Low-level x86-32 assembly rendering routines for the ROTT game engine. Handles VGA video mode setup, screen clearing, and vertical post (wall column) drawing with pixel scaling and shading lookups.

## Core Responsibilities
- Program VGA hardware into 240-pixel-height mode (SetMode240_)
- Clear visibility array and screen buffer (RefreshClear_)
- Draw unmasked wall posts at varying heights and scales (DrawPost_)
- Optimized post rendering with fractional pixel scaling (DrawHeightPost_, DrawMenuPost_, DrawMapPost_)
- Apply shading via lookup tables during scanline rendering
- Handle ceiling/floor fill at screen edges

## External Dependencies
**Notable includes/macros:**
- `.386p` — 32-bit 386+ instruction set
- `.model flat` — flat memory model (single segment)
- `IDEAL` — TASM pseudo-op for ideal syntax block
- SETFLAG macro — appears to be `test ecx, ecx` or similar

**Defined elsewhere (extern symbols):**
- `_spotvis` — visibility/spotting array (128×128)
- `_viewwidth`, `_viewheight` — viewport dimensions
- `_bufferofs` — framebuffer base offset
- `_fandc` — "fan draw ceiling" flag
- `_ylookup` — row offset lookup table
- `_centery` — viewport center Y coordinate
- `_shadingtable` — palette/shading color translation table
- `_hp_startfrac`, `_hp_srcstep` — fractional scaling parameters

# rott/rt_dr_a.h
## File Purpose
Header file declaring low-level drawing functions for the software renderer. Exposes optimized assembly routines (`RT_DR_A.ASM`) for column-based rasterization, which form the core pixel-writing stage of the rendering pipeline.

## Core Responsibilities
- Declare graphics mode setup (`SetMode240`)
- Define column drawing primitives for wall rendering (`DrawPost`, `DrawHeightPost`)
- Expose specialized drawing functions for walls, menus, and maps
- Manage refresh buffer clearing
- Define parameter passing conventions and register usage for assembly routines

## External Dependencies
- All functions defined elsewhere: `rt_dr_a.asm` (assembly implementations)
- No other includes or external symbols visible in this header

# rott/rt_draw.c
## File Purpose

Core 3D rendering engine implementing raycasting walls and scaled sprite drawing. Manages coordinate transformations, lighting, double-buffering, and special visual effects for a software-rendered first-person view.

## Core Responsibilities

- **Coordinate transformation**: Convert world 3D coordinates to 2D screen space via perspective projection
- **Wall rendering**: Raycasting-based column-by-column wall drawing with texture interpolation
- **Sprite rendering**: Draw scaled actors and static objects with depth sorting
- **Lighting/shading**: Apply distance-based fog, light sources, and fullbright effects to surfaces
- **Double-buffering & page flipping**: Manage video memory swapping with screen shake effects
- **Weapon/UI rendering**: Draw first-person weapon and status elements
- **Special effects**: Rotation, zoom, screen transitions, fades, title sequences
- **Door/pushwall rendering**: Interpolate multi-level masked wall textures

## External Dependencies

- **Math**: `FixedMul`, `FixedDiv2`, `FixedScale`, `FixedMulShift` (watcom.h pragmas)
- **Raycasting core**: `Refresh()`, `R_DrawWallColumn()`, `R_DrawColumn()` (defined elsewhere, likely rt_fc_a.h / assembly)
- **Sprite rendering**: `ScaleShape`, `ScaleTransparentShape`, `ScaleSolidShape`, `ScaleMaskedPost`, `ScaleWeapon` (rt_scale.h or rt_dr_a.h)
- **Resource management**: `W_CacheLumpName`, `W_GetNumForName`, `W_CacheLumpNum` (w_wad.h)
- **Memory**: `SafeMalloc`, `SafeFree` (z_zone.h)
- **Video**: `VGAWRITEMAP`, `VGAREADMAP`, `VGAMAPMASK`, `OUTP` (modexlib.h, VGA hardware control)
- **Camera/actors**: `player`, `firstactive`, `firstactivestat`, `objtype`, `statobj_t` (rt_actor.h)
- **Map data**: `tilemap`, `spotvis`, `doorobjlist`, `doornum`, `pwallobjlist`, `pwallnum` (rt_main.h likely)
- **Sound/music**: `MU_StartSong`, `SD_Play` (rt_sound.h)
- **UI**: `DrawStats`, `DrawMessages`, `DrawScreenSprite`, `DrawNormalSprite` (other modules)
- **Math tables**: `sintable`, `costable`, `tantable`, `gammatable`, `colormap`, `playermaps`, `redmap` (global)
- **Lighting**: `LightSourceAt`, `lights` array (rt_floor.h or similar)
- **Input**: `UpdateClientControls` (modem/netgame support, rt_net.h)

**Assembler/Hardware I/O**:
- Direct VGA mode-X register writes via `OUTP()`
- Likely inline assembly for `FixedMul` and related operations (Watcom pragmas)
- ISR callbacks for timer (`ISR_SetTime`, from isr.h)

# rott/rt_draw.h
## File Purpose
Header file declaring the core drawing and rendering system for the ROTT 3D engine. Defines data structures for visible objects, ray-tracing variables, camera state, and provides entry points for the main 3D refresh loop, screen effects, and cinematic sequences.

## Core Responsibilities
- Define the `visobj_t` structure and visible object management (`vislist`, visibility pointers)
- Declare ray-tracing state variables (`xintercept`, `yintercept`, `mapseen`)
- Manage camera transformation state (`viewx`, `viewy`, `viewangle`, sin/cos/tan tables)
- Provide light source management via the `lights` array and macros (`LightSourceAt`, `SetLight`)
- Declare the main 3D rendering function (`ThreeDRefresh`) and frame-time logic (`CalcTics`)
- Declare screen/UI functions (title screens, cinematics, credits, screen saver)
- Expose display control functions (`FlipPage`, `TurnShakeOff`)

## External Dependencies
- Standard C types and constants (e.g., `MAPSIZE`, `FINEANGLES`, `FINEANGLEQUAD`, `MAXVISIBLE` — defined elsewhere)
- Game engine fundamental types: `fixed` (fixed-point), `byte`, `word`
- Likely includes from a main header (rt_main.h or equivalent) for type definitions and map constants

# rott/rt_eng.asm
## File Purpose
Hand-optimized x86 32-bit assembly implementing ray-casting visibility detection for a tile-based game engine. Determines line-of-sight by traversing a grid, marking visible tiles and detecting opaque obstacles (walls/doors).

## Core Responsibilities
- Implement tight ray-casting loop for performance-critical visibility checks
- Traverse tile grid in two directions (X-major, Y-major stepping)
- Mark tiles as visible in spotvis array during traversal
- Call opaqueness checker (IsOpaque_) when non-empty tile encountered
- Handle two casting modes: standard line-of-sight and door-piercing variant
- Write final ray offset to _rc_off on termination

## External Dependencies
- **IsOpaque_**: External function (defined elsewhere); tests if tile index is vision-blocking
- **_spotvis, _mapseen, _tilemap, _rc_off**: External globals (defined elsewhere); visibility and map state

# rott/rt_eng.h
## File Purpose
Public header declaring the core raycasting function. This appears to be the main rendering engine interface for a raycaster-based game engine, exposing the primary ray-casting routine used for rendering 2D/3D views.

## Core Responsibilities
- Declare the `RayCast` function interface
- Specify x86 register calling conventions via pragma directive for performance-critical rendering code
- Provide public access to the raycasting engine entry point

## External Dependencies
- None declared; implementation expected in `rt_eng.c` or similar compiled object file.

# rott/rt_error.c
## File Purpose
Manages DOS-level error handling for the ROTT engine, including hard disk errors, device errors, and division-by-zero exceptions. Provides user-facing error dialogs with retry/abort options and installs interrupt service routines for critical hardware errors.

## Core Responsibilities
- Install and manage DOS hard error handler (`_harderr`)
- Intercept and handle division-by-zero (INT 0x00) exceptions
- Display formatted error messages in a windowed UI or via console
- Parse DOS device error codes and present human-readable error details
- Allow user to retry or abort operations on disk/device errors
- Maintain error handler startup/shutdown lifecycle

## External Dependencies
- **System headers:** `<dos.h>`, `<errno.h>`, `<io.h>`, `<stdio.h>`, `<conio.h>`, `<stdarg.h>`, `<mem.h>`, `<ctype.h>` — DOS/DJGPP primitives.
- **Project headers:** `rt_def.h` (typedefs), `rt_str.h` (string/font functions), `rt_menu.h` (font globals, PrintX/Y), `isr.h` (keyboard state: `Keyboard[]`, `KeyboardStarted`), `rt_vid.h` (video output), `w_wad.h`, `z_zone.h`, `rt_util.h`, `modexlib.h`, `memcheck.h`.
- **External symbols (defined elsewhere):** `SetBorderColor`, `colormap` (from video subsystem), `US_MeasureStr`, `VL_Bar`, `US_CPrint` (from rt_str/rt_menu), `Keyboard[]`, `KeyboardStarted` (from isr), `Error` (fatal error function), `_harderr`, `_dos_getvect`, `_dos_setvect` (DOS/DJGPP).
- **Constants via macros:** `OUTP` (likely port I/O macro from modexlib), `CRTC_*` registers, `MAXKEYBOARDSCAN`.

# rott/rt_error.h
## File Purpose
Public interface for the error management system in the Rise of the Triad engine. Provides initialization/shutdown routines and exposes a global flag for tracking division-by-zero errors at runtime.

## Core Responsibilities
- Declare global error state variables (`DivisionError`)
- Provide startup/shutdown entry points for error subsystem initialization
- Define the public API for engine error handling

## External Dependencies
- `boolean` type (defined elsewhere in engine, likely a typedef)
- No includes visible in this header; implementation file presumably includes necessary system/engine headers

# rott/rt_fc_a.asm
## File Purpose

Low-level x86-32 assembly rendering engine for texture-mapped row drawing with rotation, masking, and color translation. Provides four entry points for different pixel-drawing strategies (linear, rotated, masked-rotated, sky columns) optimized via fixed-point arithmetic and self-modifying code patches.

## Core Responsibilities

- **DrawRow_**: Linearly-interpolated texture mapping with color translation
- **DrawRotRow_**: Rotated texture mapping with bounds checking (x: [0–511], y: [0–255])
- **DrawMaskedRotRow_**: Rotated texture mapping with per-pixel masking (skip 0xFF values)
- **DrawSkyPost_**: Vertical sky/background column rendering with shading table lookup
- Code patching: Self-modifying code to inject runtime parameters (xstep, ystep) into instruction immediates
- Pixel-pair loop unrolling for throughput optimization

## External Dependencies

- **Includes:** `<indirect>` — Turbo Assembler (TASM) directives (`.386P`, `.MODEL`, `IDEAL` block syntax).
- **External symbols:** `_mr_xstep`, `_mr_ystep`, `_mr_xfrac`, `_mr_yfrac`, `_mr_rowofs`, `_mr_count`, `_mr_dest`, `_shadingtable` — defined elsewhere, presumed set by C caller before invocation.
- **Implicit:** Framebuffer layout (destination array layout), texture format (byte-indexed pixels), and shading table format (256-entry byte LUT) inferred from usage.

# rott/rt_fc_a.h
## File Purpose
Header file declaring low-level assembly functions for floor, ceiling, and sky rendering in the software ray-casting engine. Defines calling conventions and parameter mappings for optimized pixel-drawing routines written in x86 assembly.

## Core Responsibilities
- Declare floor/ceiling row drawing functions
- Declare sky post (vertical column) rendering function
- Declare rotation-based row drawing variants (standard and masked)
- Define Watcom C calling convention mappings via `#pragma aux` directives

## External Dependencies
- None visible (pure declarations)
- Implementations defined elsewhere (likely `rt_fc_a.asm`)
- Watcom C `#pragma aux` indicates x86 calling convention for OpenWatcom/Borland C compilers

# rott/rt_film.c
## File Purpose
Cinematic/film playback engine for Rise of the Triad. Parses script files describing timed cutscene events, manages active sprites and backgrounds, and renders animated sequences with scaling, scrolling, palette changes, and fade effects.

## Core Responsibilities
- Parse cinematic script files (`.ms` format) into event queues
- Manage event activation and timing during playback
- Allocate and free actor instances (active event instances)
- Render backgrounds, backdrops, sprites with scaling and scrolling
- Update sprite positions, scales, and lifetimes each frame
- Drive main cinematic playback loop with input handling

## External Dependencies
- `w_wad.h`: WAD resource loader (W_CacheLumpName, W_GetNumForName, W_CacheLumpNum, W_LumpLength)
- `scriplib.h`: Script tokenizer (GetToken, token, script_p, scriptend_p, endofscript, tokenready, ParseNum defined elsewhere)
- `f_scale.h`: Column renderer (R_DrawFilmColumn with register pragma, DrawFilmPost)
- `rt_scale.h`: Fixed-point math and ylookup table
- `z_zone.h`: Memory management (SafeMalloc, SafeFree)
- `rt_vid.h`: Video I/O (VL_NormalizePalette, SwitchPalette, VL_FadeOut, VL_ClearVideo, bufferofs, FlipPage, CalcTics)
- `rt_in.h`: Input polling (IN_ClearKeysDown, IN_GetMouseButtons, LastScan, tics)
- `modexlib.h`: VGA mode (VGAWRITEMAP macro for plane selection)
- `rt_util.h`: Utilities (Error, tics)
- `lumpy.h`: Image structures (patch_t with collumnofs/topoffset, lpic_t with width/height)

# rott/rt_film.h
## File Purpose
Public interface for the film/movie playback system. Declares a single function to initiate movie playback and exposes a global variable for display center positioning during movie rendering.

## Core Responsibilities
- Declare the public movie playback entry point
- Export display center Y coordinate for screen positioning
- Provide minimal abstraction for cinematic/demo sequences

## External Dependencies
- Implementation provided by rt_film.c
- `dc_ycenter` defined elsewhere in codebase

# rott/rt_floor.c
## File Purpose
Implements floor, ceiling, and parallax sky rendering for the 3D raycaster engine. Handles texture mapping, light shading, and horizontal span rasterization for the ground and overhead planes in the game world.

## Core Responsibilities
- Initialize and manage floor/ceiling/sky texture data at level load
- Render the main floor and ceiling planes via horizontal scanline traversal
- Implement parallax-scrolling sky rendering with configurable horizon height
- Calculate per-pixel light levels based on distance (fog/brightness)
- Manage VGA plane switching for chunked 256-color graphics
- Load and process texture lumps from the WAD resource system

## External Dependencies
- **WAD Loading:** `W_CacheLumpNum()`, `W_GetNumForName()` (w_wad.h)
- **Memory:** `SafeMalloc()`, `SafeFree()` (z_zone.h)
- **Engine globals:** `viewangle`, `viewx`, `viewy`, `viewsin`, `viewcos`, `player`, `posts[]`, `pixelangle[]`, `bufferofs`, `ylookup[]`, `MAPSPOT()` (defined elsewhere)
- **Rendering:** `shadingtable`, `colormap`, `greenmap`, `basemaxshade`, `lightninglevel`, `fog`, `fulllight` (rt_view.h, rt_draw.h)
- **VGA I/O:** `VGAWRITEMAP()`, `VGAMAPMASK()` (modexlib.h)
- **Assembly functions:** `DrawSkyPost()`, `DrawRow()` (rt_fc_a.h, rt_fc_a.asm)
- **Fixed-point math:** `FixedMulShift()` (watcom.h)
- **Constants:** `MAXVIEWHEIGHT`, `MAXSKYSEGS`, `MAXSKYDATA`, `MINSKYHEIGHT`, `FINEANGLES` (_rt_floo.h, rt_def.h)

# rott/rt_floor.h
## File Purpose
Public header declaring the interface for floor, ceiling, and sky rendering functionality in the ray-casting renderer. Exposes ray-marching state variables and plane-drawing entry points needed by the rendering pipeline.

## Core Responsibilities
- Declare floor and ceiling plane rendering functions
- Expose ray-marching step/fraction state for coordinate calculations
- Provide sky rendering API (generation, drawing, visibility checks)
- Manage parallax sky enable/disable flag

## External Dependencies
- **Includes/Imports:** None visible (this is a public header; implementation in rt_floor.c).
- **External symbols:** Basic C types only (`int`, `byte`, `boolean`, `void`); implementations defined elsewhere.

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

## External Dependencies
- **rt_actor.h**: `objtype` (actor/entity structure), `AlternateInformation`, `exit_t` (level exit codes), enemy and weapon class definitions
- **lumpy.h**: `pic_t` (picture/sprite header)
- **rt_cfg.h**: `AlternateInformation` (resource path/availability)
- **rt_playr.h**: `playertype` (player state structure)
- **Other:** Sound and rendering systems (e.g., `GameMemToScreen()` implies framebuffer/palette API)

# rott/rt_in.c
## File Purpose

Input manager for ROTT that handles detection, initialization, and polling of multiple input devices (keyboard, mouse, joysticks, SpaceBall, Cyberman, Assassin). Provides normalized control reading and text input queuing for chat/menus.

## Core Responsibilities

- Initialize and detect connected input devices (keyboard, mouse, joysticks, special hardware)
- Process keyboard queue from ISR and maintain keyboard state array
- Convert raw joystick/mouse input to normalized control directions and magnitudes
- Calibrate and scale joystick analog input with threshold-based dead zones
- Provide mouse button and joystick button state to game logic
- Handle acknowledgment loops for menu waits (wait for key/button release then press)
- Queue ASCII character input for chat messages and modem text composition
- Integrate special input devices (SpaceBall, Cyberman, Assassin) via SWIFT protocol
- Shutdown and cleanup input subsystems on exit

## External Dependencies

**Hardware/OS interfaces:**
- `<dos.h>`, `<i86.h>`: DOS real-mode interrupt and port I/O
- `int386()`: Watcom C function to invoke real-mode INT (mouse INT 33h, etc.)
- `inp()`: Port I/O read (joystick at port 0x201, keyboard at port 0x60)
- ISR keyboard queue (`Keyhead`, `Keytail`, `KeyboardQueue[]`) defined in `isr.h`

**Game subsystems (defined elsewhere):**
- `OpenSpaceBall()`, `GetSpaceBallButtons()`, `CloseSpaceBall()` — SpaceBall device (`rt_spball.h`)
- `SWIFT_Initialize()`, `SWIFT_GetDynamicDeviceData()`, `SWIFT_TactileFeedback()`, `SWIFT_Terminate()` — Cyberman/Assassin 3D controller (`rt_swift.h`)
- `JoyStick_Vals()` — Joystick port read (implementation in `_rt_in.c` or asm)
- `FinishModemMessage()`, `UpdateModemMessage()`, `ModemMessageDeleteChar()`, `AddMessage()` — Modem chat subsystem (`rt_com.h`, `rt_net.h`)
- `US_CheckParm()` — Utility string param check (`rt_util.h`)
- Global game state: `gamestate`, `consoleplayer`, `numplayers`, `CommbatMacros[]`, `Messages[]` (from `rt_playr.h`, `rt_msg.h`, etc.)
- Configuration globals: `mouseenabled`, `spaceballenabled`, `cybermanenabled`, `assassinenabled`, `quiet` (from various headers)

# rott/rt_in.h
## File Purpose
Public input system header for RT_IN.C. Defines interfaces for polling keyboard, mouse, joystick, and networking/modem input, translating hardware signals into unified control structures for player movement and actions.

## Core Responsibilities
- Define input state data structures (CursorInfo, ControlInfo, keyboard/joystick definitions)
- Declare device initialization/shutdown for keyboard, mouse, joystick
- Provide control polling and translation (scan code → motion/direction)
- Support multi-input modes (keyboard, mouse, joystick, network play)
- Manage input queue state (letter queue for text entry)
- Define modem message structures for networked play

## External Dependencies
- **develop.h:** Build configuration flags (SHAREWARE, SUPERROTT, WEAPONCHEAT, etc.)
- **rottnet.h:** Networking constants (MAXPLAYERS, MAXNETNODES) and rottcom_t structure for driver communication
- **Implicit:** Assumes byte, word, boolean, int types and interrupt-driven keyboard handler (LastScan volatile)

# rott/rt_main.c
## File Purpose

This is the main entry point and core game loop orchestrator for Rise of the Triad. It initializes the engine (memory, input, sound, video), implements the primary state machine, and manages gameplay, menu, and cinematic transitions. Also handles command-line parsing, debugging, and auxiliary features like demo recording and screen capture.

## Core Responsibilities

- **Game initialization**: Set up memory manager, input system, sound/music, video mode, WAD files, palette/font cache
- **State machine**: Manage transitions between titles, gameplay, death, level completion, demos, and cinematics
- **Main game loop**: Orchestrate frame updates, rendering, and input polling during active play
- **Pause handling**: Implement pause state with optional screensaver
- **Input/keyboard**: Poll player input, handle debug keys, remote ridicule commands, and volume adjustments
- **Command-line parsing**: Interpret launch parameters (warp, turbo, sound setup, time limits, etc.)
- **Configuration**: Read/write game config, manage display settings, difficulty, and player state
- **Cinematics & demos**: Load cinematics, record/play demos, handle transitions
- **Screen capture**: Save gameplay screenshots in LBM or PCX format

## External Dependencies

### Notable Includes / Imports
- **Engine core:** rt_def.h (constants, types), rt_actor.h (actor management), rt_stat.h (static objects), rt_vid.h (video/palette), rt_game.h (game state)
- **Rendering:** rt_draw.h, rt_view.h, rt_scale.h (3D rendering), modexlib.h (VGA mode X), rt_dr_a.h
- **Input:** rt_in.h (input polling), isr.h (interrupt handlers)
- **Audio:** music.h, fx_man.h, rt_sound.h (sound/music control)
- **Game systems:** rt_door.h (doors), rt_floor.h (elevators), rt_playr.h (player state), rt_menu.h (menus), rt_map.h (map/level), rt_ted.h (TED editor), rt_net.h, rottnet.h (networking), cin_main.h (cinematics)
- **Utilities:** z_zone.h (memory manager), w_wad.h (WAD loader), rt_util.h, rt_msg.h (messages), rt_cfg.h (config), rt_rand.h (RNG), rt_debug.h (debug macros)
- **Platform:** dos.h, conio.h, graph.h, direct.h, bios.h, process.h, fcntl.h, io.h, malloc.h (DOS/Watcom SDK)
- **Data:** scriplib.h (script parsing), develop.h, version.h, memcheck.h (memory debugging)

### Symbols Defined Elsewhere
- **Game state:** gamestate (global), locplayerstate, PLAYERSTATE, player, consoleplayer, numplayers
- **Input:** LastScan, Keyboard[], buttonpoll[], buttonscan[]
- **Rendering:** bufferofs, displayofs, origpal, ylookup[]
- **Audio:** MUvolume, FXvolume, MusicMode, FXMode
- **Battle:** BATTLE_Options, BATTLE_ShowKillCount, BATTLE_Init, BATTLE_CheckGameStatus, BATTLEMODE
- **Demo/Demo state:** demorecord, demoplayback, demo-loading functions
- **Level data:** mapplanes, tilemap, TRIGGER[], Clocks[], firstactive, firstemptystat, etc.

# rott/rt_main.h
## File Purpose
Main engine header defining the global game state (`gametype`), core initialization/update/shutdown functions, and central game variables. Acts as the primary interface for the game's main loop and state management.

## Core Responsibilities
- Define the central game state structure tracking score, kills, inventory, difficulty, and battle settings
- Declare lifecycle functions for game initialization, updates, and shutdown
- Export global debugging and development mode flags
- Manage special power-up timer state
- Track game version information (shareware vs. registered)
- Expose main loop control functions (pause, cinematic playback, screen saving)

## External Dependencies
- **develop.h** — Development flags (DEBUG, SHAREWARE, WEAPONCHEAT, etc.)
- **rt_def.h** — Engine constants (screen geometry, angles, map sizes, actor limits, global flags)
- **rottnet.h** — Networking primitives (MAXPLAYERS, rottcom_t structure)
- **rt_battl.h** — Battle system types (battle_type, battle_status)

**Defined elsewhere:** Function implementations, actor/sprite systems, rendering backend, input handling, audio subsystem.

# rott/rt_map.c
## File Purpose
Implements interactive map/minimap display and exploration system for Rise of the Triad. Provides zoom-able map views showing level layout, explored areas, actors, sprites, and player position with directional indicator.

## Core Responsibilities
- Render minimap at configurable zoom levels with proper tile scaling
- Draw full-screen map at maximum zoom showing entire level
- Track and visualize explored/seen areas of the map
- Render all map element types (walls, doors, animated walls, masked objects, actors, sprites)
- Handle interactive map mode with keyboard navigation and zoom controls
- Manage map color schemes and display options
- Scale and position sprites appropriately on minimap

## External Dependencies
- Notable includes: `rt_def.h` (common types), `rt_draw.h`, `rt_dr_a.h` (low-level drawing), `w_wad.h` (lump caching), `rt_door.h`, `modexlib.h` (VGA), `rt_vid.h`, `rt_in.h` (input), `z_zone.h` (memory)
- Defined elsewhere: `tilemap[][]`, `actorat[][]`, `sprites[][]`, `doorobjlist[]`, `maskobjlist[]`, `animwalls[]`, `mapseen[][]`, `player` (global), `bufferofs`, `ylookup[]`, `egacolor[]`, `sky`, `mapwidth`, `mapheight`, `shapestart`, `Keyboard[]`, `RandomNumber`, `W_CacheLumpNum`, `IsPlatform`, `IsWindow`, `DrawPositionedScaledSprite`, `VL_DrawLine`, `VL_ClearBuffer`, `FlipPage`, `CalcTics`

# rott/rt_map.h
## File Purpose
Public interface header for map rendering and display functionality in the Rise of the Triad engine. Declares the main map display function and a cheat code handler for map visualization.

## Core Responsibilities
- Export the primary map rendering function (`DoMap`)
- Export the map cheat code handler (`CheatMap`)
- Provide a minimal public API for map-related operations from other game modules

## External Dependencies
- None visible (this is a pure interface header with no includes)
- Implementation and callers defined elsewhere in the codebase

## External Dependencies
- **Input**: `rt_in.h` — `IN_UpdateKeyboard()`, `IN_ReadControl()`, `IN_GetMouseButtons()`, `INL_GetJoyDelta()`, `IN_JoyButtons()`, `CalibrateJoystick()`, keyboard scan code arrays
- **Resources**: `w_wad.h` — `W_GetNumForName()`, `W_CacheLumpNum()`, `W_CacheLumpName()`, `W_GetNameForNum()`
- **Graphics**: `rt_draw.h`, `rt_view.h`, `rt_vid.h` — `VWB_DrawPic()`, `VW_UpdateScreen()`, `VL_DrawPostPic()`, `DrawNormalSprite()`, `VL_FadeOut()`, `VL_FillPalette()`, screen coordinate macros
- **Sound**: `rt_sound.h` — `SD_Play()`, `SD_Startup()`, `SD_Shutdown()`, `MU_Startup()`, `MU_Shutdown()`, `MU_FadeOut()`, sound enumeration constants
- **Game State**: `rt_main.h` — `gamestate`, `playstate`, `locplayerstate`, `consoleplayer`, `numplayers`, `modemgame`
- **Game Logic**: `rt_game.h` — `BATTLE_SetOptions()`, `BATTLE_GetOptions()`, `GamePaused`, `RefreshPause`
- **Config**: `rt_cfg.h` — `WriteConfig()`, `GetPathFromEnvironment()`, `GetSavedMessage()`, `GetSavedHeader()`, `LoadTheGame()`, `SaveTheGame()`, various global settings
- **Utilities**: `z_zone.h` (memory), `rt_util.h`, `rt_str.h`, `rt_scale.h`, `rt_com.h`, `lumpy.h` (graphics headers), `modexlib.h` (VGA mode X)
- **Sound Hardware**: `fx_man.h` — sound device setup and initialization; DOS/system headers for hardware I/O and file operations

# rott/rt_menu.h
## File Purpose
Public header for the menu and control panel system in Rise of the Triad. Declares all menu structures, UI rendering functions, game state navigation (main menu, options, load/save), and multiplayer setup handlers.

## Core Responsibilities
- Menu rendering and item management (fonts, positioning, active states, texture display)
- Game state transitions (main menu → new game, load, settings, battle modes, quit)
- Player selection, name entry, and multiplayer configuration (teams, CTF, modems)
- Save/load game functionality including quick-save/undo
- Sound and audio options configuration
- Color selection and display options (detail level, bobbing, flip speed)
- Input capture and menu navigation (keyboard scanning, hotkeys, multi-page menus)
- Screen resource allocation/deallocation for menu rendering

## External Dependencies
- **lumpy.h** — pic_t, lpic_t, font_t, cfont_t, patch_t, transpatch_t (graphics structures)
- **rt_in.h** — ControlInfo, KeyboardDef, JoystickDef, Motion, Direction, ControlType enums; input query functions (IN_ReadControl, IN_WaitForKey, etc.)
- **Undefined here** — boolean, byte, word (primitive types defined elsewhere, likely develop.h or standard headers); file I/O functions (defined elsewhere); rendering functions; game state globals

# rott/rt_msg.c
## File Purpose
Implements the on-screen message and notification system for Rise of the Triad. Manages creation, display timing, rendering, and lifecycle of temporary notifications (pickups, cheats, system alerts) and permanent UI messages (multiplayer chat, player selection menus). Handles color-coding by message type and background restoration for erased message regions.

## Core Responsibilities
- Initialize and reset the message system
- Manage message queue with fixed-size array and slot reuse
- Add, delete, and update messages with type-based priority
- Sort and order messages for display
- Render messages with color-coding based on message flags (type)
- Update message timers and auto-expire non-permanent messages each frame
- Handle modem/multiplayer chat input and character editing
- Render player selection menus for directed messaging
- Restore screen background after messages are erased

## External Dependencies
- **rt_def.h** — Core engine types and constants (MAXMSGS, boolean, byte, memset, Error)
- **rt_view.h** — Display/viewport utilities (SHOW_TOP_STATUS_BAR, YOURCPUSUCKS_Y, fontcolor, egacolor[])
- **z_zone.h** — Memory allocation (SafeMalloc, SafeFree)
- **w_wad.h** — Resource caching (W_CacheLumpName, "backtile", "ifnt")
- **lumpy.h** — Graphics structures (pic_t, cfont_t)
- **rt_vid.h** — Rendering primitives (DrawIString, DrawTiledRegion, DrawCPUJape)
- **rt_com.h** — Modem/network communication (COM_MAXTEXTSTRINGLENGTH, MSG global struct)
- **rt_net.h** / **rt_playr.h** — Multiplayer support (numplayers, consoleplayer, PLAYERSTATE[], gamestate.teamplay)
- **rt_main.h** — Game state (GamePaused, ticcount, quiet)
- **rt_menu.h**, **rt_str.h** — Menu and string utilities
- **Standard C** — mem.h (memset, memcpy), stdlib.h

---

**Notes:**
- Message display is soft-erased (background restored over 3 frames) rather than cleared instantly, reducing visual flicker.
- Message ordering is re-computed after each add/delete to maintain sort invariant.
- Permanent vs. temporary distinction via macro `PERMANENT_MSG(flags)` affects memory allocation size and timer behavior.

# rott/rt_msg.h
## File Purpose
Header for the in-game message/dialog system in Rise of the Triad. Defines message types, priority flags, and the message queue infrastructure for displaying game events, system messages, modem communication, and cheat notifications.

## Core Responsibilities
- Define message type constants and flag bits (priority, permanence, deletion policy)
- Declare the global message queue and enable/disable toggle
- Provide message lifecycle functions (add, delete, render, clear)
- Support modem/network message input and editing
- Handle message background restoration and priority-based deletion

## External Dependencies
- `byte`, `boolean`, `int`, `char *` — standard C types (defined elsewhere, likely `rt_types.h`)
- No explicit external module dependencies in this header; message text likely allocated by caller

# rott/rt_net.c
## File Purpose
Implements network command synchronization, packet routing, and game state management for ROTT's multiplayer mode. Handles the critical game loop for both client and server sides, including control polling, packet buffering, loss recovery, and demo recording/playback.

## Core Responsibilities
- **Command management**: Allocate/deallocate per-player command queues; track packet arrival status
- **Client-side control**: Poll input, advance control time, format and send movement packets
- **Server-side aggregation**: Collect client packets, broadcast aggregated server packet
- **Packet routing**: Dispatch incoming packets by type to appropriate handlers
- **Reliability**: Resend lost packets via COM_REQUEST/COM_FIXUP protocol
- **Synchronization**: Time-sync clients, verify game state consistency (optional SYNCCHECK)
- **Demo system**: Record/playback gameplay for automated testing or demonstration
- **Player state application**: Unpack network commands into actor momentum and buttons

## External Dependencies
- **I/O:** `<dos.h>`, `<fcntl.h>`, `<io.h>`; ReadPacket, WritePacket (rt_com.h)
- **Memory:** SafeMalloc, SafeLevelMalloc, SafeFree (z_zone.h)
- **Timer:** CalcTics, ISR_SetTime (isr.h); ticcount, oldtime
- **Input:** PollControls, INL_GetMouseDelta, Keyboard[] (rt_util.h)
- **Game state:** gamestate, PLAYERSTATE[], PLAYER[], consoleplayer, characters[]
- **Sound:** SD_Play, SD_GetSoundData (rt_sound.h)
- **Battle:** BATTLE_GetOptions, AssignTeams, BATTLE_Team[] (rt_battl.h)
- **Debug:** Error, SoftError (rt_debug.h); AbortCheck
- **Network:** rottcom, ROTTpacket[], MAXCOMBUFFERSIZE (rottnet.h)

# rott/rt_net.h
## File Purpose
Network protocol and command management header for ROTT's multiplayer system. Defines packet structures, command types, and function interfaces for synchronizing game state, player input, and demo recording across networked game instances (both modem and LAN).

## Core Responsibilities
- Define network command protocol (25+ command types: delta, sync, pause, respawn, text messages, etc.)
- Declare packet structures for all network message types (player descriptions, game config, synchronization)
- Manage demo recording/playback system integration with network
- Coordinate client-side input capture and server-side distribution
- Handle player description and game configuration exchange at startup
- Implement remote ridicule (voice/text) transmission between players
- Provide synchronization checks and timeout mechanisms

## External Dependencies
- **develop.h** – feature flags (SYNCCHECK, etc.)
- **rottnet.h** – low-level network driver interface (rottcom_t, modem/network game constants)
- **rt_actor.h** – object/actor structures (objtype, classtype)
- **rt_battl.h** – battle system types (battle_type, specials, battle_options)
- **rt_playr.h** – player state (playertype, MAXCODENAMELENGTH)
- **rt_main.h** – main game loop integration (VBLCOUNTER timing)
- **Defined elsewhere:** External symbols `VBLCOUNTER` (VBL tick counter), `MAXPLAYERS` (from rottnet.h), game state globals

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

# rott/rt_playr.h
## File Purpose
Header file defining player state, movement, input handling, and weapon systems for the ROTT game engine. Declares interfaces for player spawning, control polling, combat mechanics, and multiplayer support.

## Core Responsibilities
- Define player state structure (health, weapons, position, input state)
- Declare player input polling functions (keyboard, mouse, joystick, special devices)
- Manage player movement, collision, and physics
- Control weapon selection, firing, and item acquisition
- Define attack sequences and weapon configuration data
- Support character selection and statistics
- Interface with network multiplayer system
- Track dead players and respawn mechanics

## External Dependencies
- **Includes**: rt_actor.h (objtype, classtype), rt_stat.h (statobj_t), states.h (statetype), rottnet.h (multiplayer), rt_battl.h (battle mode), develop.h (build flags)
- **Defined elsewhere**: objtype, statetype, statobj_t, classtype, thingtype, exit_t, dirtype (from included headers)
- **Notable globals**: BATTLEMODE (battle mode flag), firstactive/lastactive (actor list pointers), angletodir[] (angle lookup)

# rott/rt_rand.c
## File Purpose
Implements a deterministic pseudo-random number generator using a pre-computed lookup table of 2048 values. Provides two independent RNG streams for game logic and other subsystems (e.g., sound), with support for debug logging and state inspection for replay/record functionality.

## Core Responsibilities
- Initialize RNG system at engine startup with time-based seeds
- Provide two independent random value streams (GameRNG and RNG)
- Manage circular-buffer indices for deterministic playback
- Support optional debug mode that logs all RNG calls with source information
- Allow external query and manual manipulation of RNG indices for testing/replay

## External Dependencies
- **System includes**: `<time.h>` (time function)
- **Local headers**: `rt_def.h`, `_rt_rand.h` (RandomTable, SIZE_OF_RANDOM_TABLE), `rt_rand.h`, `develop.h` (RANDOMTEST, DEVELOPMENT flags), `rt_util.h` (SoftError macro), `memcheck.h`
- **Defined elsewhere**: RandomTable (2048 pre-computed byte values in _rt_rand.h), SoftError (macro from rt_util.h), SIZE_OF_RANDOM_TABLE constant

# rott/rt_rand.h
## File Purpose
Public interface for the random number generator system. Declares initialization, seeding, and RNG functions with conditional debug logging support controlled by the `RANDOMTEST` compile flag.

## Core Responsibilities
- Declare RNG initialization and seed management functions
- Provide `GameRNG()` and `RNG()` function wrappers with macro aliases
- Support debug mode (RANDOMTEST) that logs RNG calls with string labels and values
- Expose RNG state index management (get/set)
- Abstract production vs. debug signatures behind preprocessor conditionals

## External Dependencies
- `develop.h` — provides `RANDOMTEST` and `RANDOMTEST` compile flags

# rott/rt_sc_a.asm
## File Purpose
Low-level x86-32 assembly implementation of column rasterization routines for raycasting-based 3D rendering. Contains five specialized column drawing functions that perform texture mapping, color translation, and pixel writes to the framebuffer with different visual effects (textured, solid, transparent, clipped, high-precision wall).

## Core Responsibilities
- Rasterize vertical columns from texture sources to screen memory with fixed-point texture coordinate interpolation
- Apply color translation via shading lookup tables for lighting/palette effects
- Optimize rendering via pair-processing (2 pixels per loop iteration) and self-modifying code
- Support multiple rendering variants: textured, solid-fill, transparent, clipped bounds, and high-precision wall textures
- Manage screen address calculation and vertical iteration

## External Dependencies
- **Includes/defines:** `.386`, `.MODEL flat` (flat memory model, 32-bit addressing).
- **External symbols referenced:**
  - _centery, _centeryclipped (DWORD) — camera Y position
  - _dc_yl, _dc_yh (DWORD) — column Y bounds
  - _dc_iscale (DWORD) — inverse scale for texture coordinate step
  - _dc_texturemid (DWORD) — base texture coordinate offset
  - _ylookup (DWORD array) — Y-to-screen-offset LUT
  - _dc_source (DWORD) — texture bitmap pointer
  - _shadingtable (DWORD) — color translation LUT

# rott/rt_sc_a.h
## File Purpose
Header file declaring low-level assembly drawing functions for rendering masked columns (vertical screen strips) during the ROTT rendering pipeline. Provides optimized scanline drawing primitives with explicit register-level calling conventions for x86 assembly implementations.

## Core Responsibilities
- Declare column/scanline drawing functions for masked sprites and walls
- Specify x86 calling conventions and register mappings via `#pragma aux` directives
- Define the interface between C renderer and assembly-optimized drawing code
- Support solid color fills and transparent/translucent column rendering

## External Dependencies
- Standard C `byte` type (unsigned char)
- Borland C++ `#pragma aux` compiler directive for custom calling conventions
- x86 register model (eax, ebx, ecx, edx, esi, edi)

# rott/rt_scale.c
## File Purpose
Handles scaling, transformation, and rendering of sprites in the ROTT software-rasterized 3D engine. Implements column-based sprite drawing with support for transparency, masking, light shading, and variable scaling for perspective-correct rendering.

## Core Responsibilities
- Calculate and apply light/shading tables based on player position and object height
- Scale and render individual sprite columns (posts) at various transparency/masking modes
- Composite scaled sprite shapes into the VGA framebuffer with occlusion testing
- Render weapon/HUD sprites with custom scaling
- Support both scaled and unscaled sprite drawing for UI elements
- Handle VGA planar mode rendering across multiple pixel planes

## External Dependencies
- **rt_draw.h** — `visobj_t`, shading table, light lookup macros, drawing externs
- **rt_def.h** — Screen/engine constants (FINEANGLES, SFRACBITS, SFRACUNIT, HEIGHTFRACTION, VIEWWIDTH, VIEWHEIGHT)
- **watcom.h** — `FixedMul()`, fixed-point arithmetic (Watcom inline assembly)
- **modexlib.h** — VGA register macros (`VGAWRITEMAP()`, `VGAMAPMASK()`, `VGAREADMAP()`), framebuffer pointers (`bufferofs`, `ylookup[]`)
- **w_wad.h** — `W_CacheLumpNum()` (resource loading)
- **z_zone.h** — Memory zone manager (for `PU_CACHE` flag)
- **rt_scale.h** — Function declarations
- **_rt_scal.h**, **rt_sc_a.h** — Assembly stubs or macro extensions
- **engine.h**, **rt_main.h**, **rt_ted.h**, **rt_vid.h**, **rt_view.h**, **rt_playr.h** — Global engine state (player, MISCVARS, lights, posts[], etc.)

# rott/rt_scale.h
## File Purpose
Header declaring sprite scaling and 2D projection functions for the Rise of the Triad 3D renderer. Handles conversion of 3D sprite objects to screen-space, with support for scaling, transparency, and lighting effects. Critical bridge between visibility culling (rt_draw.h) and the column-based software rasterizer.

## Core Responsibilities
- Declare scaling entry points for 3D sprites to 2D screen projection
- Export vertical column (post) rendering functions for scaled sprites
- Manage transparency and masking parameters during rasterization
- Provide weapon and HUD sprite drawing functions
- Handle lighting level calculation based on sprite depth/height
- Expose global scaling state (inverse scale, texture coordinates, clipping bounds)

## External Dependencies
- **rt_draw.h**: visobj_t structure, global projection state (viewx, viewy, viewangle, lights), frame variables (tics, levelheight), math tables (sintable, costable, tantable)
- Implied: Texture/sprite resource manager (lump system), framebuffer manager, shading tables

# rott/rt_ser.c
## File Purpose
Implements serial/modem communication for multiplayer networking in the ROTT game engine. Manages UART (8250/16550) hardware, interrupt-driven I/O queues, and frame-based packet encoding/decoding with escape sequence handling for reliable data transmission over serial links.

## Core Responsibilities
- UART hardware initialization, configuration (baud rate, interrupt vectors), and shutdown
- Interrupt service routine (ISR) for handling serial transmit/receive/modem status interrupts
- Ring buffers (queues) for buffering incoming and outgoing serial data
- Frame-based packet protocol with escape-character encoding to handle binary data
- Interactive "talk mode" for keyboard-to-serial communication
- Queue manipulation (read/write single bytes and multi-byte buffers)

## External Dependencies
- **Includes:** `<conio.h>` (BIOS keyboard), `<dos.h>` (DOS interrupt), `<stdio.h>`, `<stdlib.h>`, `<mem.h>` (memcpy), `<bios.h>` (_bios_keybrd), `memcheck.h` (memory debugging)
- **Local headers:** `rottser.h` (serialdata_t), `_rt_ser.h` (UART register macros, que_t, internal protos), `rt_ser.h` (public API), `rt_def.h` (game constants), `rt_def.h` (define MAXPACKET via MAXPACKETSIZE)
- **Defined elsewhere:** `_dos_getvect()`, `_dos_setvect()` (DOS real-mode interrupt vectors); `inp()`, `outp()` (port I/O); `_disable()`, `_enable()` (CPU interrupt flags); `rottcom` (global config structure); `MAXPACKETSIZE` constant

# rott/rt_ser.h
## File Purpose
Serial modem communication interface for multiplayer networking. Provides setup, teardown, and packet I/O functions for players connected via serial/modem links in networked gameplay.

## Core Responsibilities
- Initialize and shutdown modem-based multiplayer game sessions
- Read incoming serial packets into a global buffer
- Write outgoing packets to the serial port
- Manage global packet data state

## External Dependencies
- `rt_def.h`: Supplies `boolean` typedef, standard types (`char`, `int`)
- `MAXPACKET`: Constant defined elsewhere (likely `rt_def.h`)
- Serial hardware/ISR: Modem interrupt handlers and driver layer (implementation not visible)

# rott/rt_sound.c
## File Purpose
Implements sound and music systems for the Rise of the Triad engine, including FX device initialization, 2D/3D sound playback with spatial audio, music song management with fading, and memory caching for audio resources.

## Core Responsibilities
- Initialize and manage sound card hardware (Sound Blaster, Adlib, UltraSound, etc.)
- Play sound effects with optional 3D positioning and pitch modulation
- Control sound panning and volume per-sound or globally
- Manage music playback with song selection, fading, and loop control
- Cache and pre-cache audio lumps for performance
- Store and restore music playback state for save/load game functionality
- Map sound and music to appropriate hardware device types based on configuration

## External Dependencies
- **Sound/FX system**: `fx_man.h` (FX_Init, FX_PlayVOC3D, FX_PlayWAV3D, FX_SetCallBack, FX_SetPitch, FX_Pan3D, FX_StopSound, FX_SoundActive)
- **Music system**: `music.h` (MUSIC_Init, MUSIC_PlaySong, MUSIC_FadeVolume, MUSIC_GetPosition, MUSIC_SetPosition, MUSIC_SongPlaying)
- **WAD/lump system**: `w_wad.h` (W_GetNumForName, W_GetNameForNum, W_CacheLumpNum, W_CacheLumpName, W_LumpLength)
- **Math**: `rt_util.h` (FindDistance, atan2_appx); inline `PitchOffset()` using RandomNumber()
- **Game state**: `rt_playr.h` (global `player` struct for relative sound positioning)
- **Utility**: `z_zone.h` (memory); `rt_rand.h` (RandomNumber); `rt_menu.h` (HandleMultiPageCustomMenu); `rt_main.h` (ticcount, Error, SoftError)
- **Configuration**: `rt_cfg.h` (FXMode, MusicMode, NumVoices, NumChannels, NumBits, FXvolume, MUvolume, stereoreversed, MidiAddress)
- **Development**: `develop.h` (DEVELOPMENT, SOUNDTEST flags for conditional logging)

# rott/rt_sound.h
## File Purpose

Public sound and music subsystem header for Rise of the Triad. Defines enums for all in-game sound effects and music, supported audio hardware types, and declares the API for sound playback, spatial audio, music management, and device initialization/shutdown.

## Core Responsibilities

- Define comprehensive enumeration of game sounds and music events
- Declare sound effect playback API (direct play, positioned, 3D, pitched)
- Declare music playback and fade control API
- Define supported audio card types (Adlib, General MIDI, Sound Blaster, etc.)
- Declare sound device setup, startup, and shutdown routines
- Provide macro wrappers for music volume/fade operations

## External Dependencies

- **music.h**: Declares `MUSIC_*` macro/function implementations; wrapped by `MU_*` functions
- **develop.h**: Debug/development feature flags (SOUNDTEST, PRECACHETEST, etc.)
- **sndcards.h**: Referenced by music.h; audio card abstraction layer (not included here)
- Audio device driver: Implied via `SD_SetupFXCard()`, hardware initialization
- Resource/file system: Implied by `SD_PreCacheSound()`, sound asset loading

**Notes:**
- 300+ `game_sounds` enum values indicate comprehensive audio asset library
- Macro wrappers (`MU_Continue()`, `MU_Pause()`, etc.) delegate to `MUSIC_*` functions for convenience
- Spatial audio (2D panning, 3D polar) suggests immersive sound design
- Support for multiple audio hardware (Adlib, MIDI, Sound Blaster) reflects 1990s hardware diversity

# rott/rt_spbal.c
## File Purpose

Manages Spaceball 6-DOF input device integration for the ROTT game engine. Handles device initialization, real-time motion polling, button mapping, and conversion of raw Spaceball data into game control inputs (movement, aiming, weapon firing).

## Core Responsibilities

- Initialize Spaceball hardware and load device configuration from file
- Poll Spaceball for raw 6D motion (translation/rotation) and button states
- Filter and normalize raw motion data (single-axis, planar, null-zone suppression)
- Apply configurable warp/sensitivity curves to each motion axis via fixed-point math
- Map physical buttons to game actions (attack, use, map, aim, pause, turbo-fire, weapon swap)
- Convert Spaceball motion into game control buffer deltas (forward/strafe/vertical)
- Implement turbo-fire accumulation mechanic by modulating rotation input
- Shut down device cleanly on exit

## External Dependencies

- **Spaceball library** (splib.h): `SpwSimpleOpen`, `SpwSimpleGet`, `SpwSimpleClose` — raw hardware I/O.
- **Spaceball config system** (_rt_spba.h, sbconfig.h): `SbConfigParse`, `SbConfigGetWarpRange`, `SbConfigGetButtonNumber` — configuration parsing and warp record retrieval.
- **Fixed-point math** (watcom.h): `FixedMul` — 16.16 fixed-point multiplication for motion scaling.
- **Game state** (rt_playr.h): `player` (global player object), `buttonpoll[]` (button polling array), `controlbuf[3]` (accumulated control delta), `PausePressed` (pause flag), angle/height constants.
- **Rendering context** (rt_draw.h): `viewcos`, `viewsin`, `costable[]`, `sintable[]` — view-relative trigonometry for strafe angle computation.
- **Utility** (rt_util.h, rt_main.h): `GetPathFromEnvironment`, `DoMap` — path resolution and map display command.
- **Math constants** (rt_def.h): `FINEANGLES`, `FL_FLEET` player flag.
- **I/O** (conio.h, io.h): `kbhit()`, `printf()` — console I/O for interactive test.

# rott/rt_spbal.h
## File Purpose
Public interface header for SpaceBall input device handling. Declares functions to initialize, poll, and shut down the SpaceBall device for reading button and positional input during gameplay.

## Core Responsibilities
- Initialize and open the SpaceBall device
- Shut down and close the SpaceBall device
- Poll SpaceBall input state each frame
- Query current SpaceBall button states

## External Dependencies
- None (header-only declarations)

# rott/rt_sqrt.h
## File Purpose
Header file declaring two fixed-point square root functions optimized for performance: a low-precision variant (8.8 bit accuracy) and a high-precision variant (8.16 bit accuracy). These are core mathematical utilities for the game engine's physics and graphics calculations.

## Core Responsibilities
- Declare fixed-point square root functions with Watcom C++ inline assembly implementations
- Provide low-precision (`FixedSqrtLP`) for speed-critical paths
- Provide high-precision (`FixedSqrtHP`) for accuracy-critical paths
- Include binary-search-based algorithms optimized for 32-bit x86 architecture

## External Dependencies
- No includes or imports.
- Uses Watcom C++ `#pragma aux` directive for inline x86 assembly embedding.
- Fixed32 type referenced in comments but not defined in this file (likely defined elsewhere).

# rott/rt_stat.c
## File Purpose
Manages static objects (items, decorations, environmental hazards, lights) in the game world. Handles spawning, removal, animation, lighting effects, and persistence/respawn mechanics for 91 distinct static object types.

## Core Responsibilities
- Spawn and remove static objects at tile coordinates with height/type parameters
- Maintain active/inactive and free object pools via doubly-linked lists
- Animate sprite frames and wall textures via per-frame updates
- Manage light sources and compute light influence on surrounding tiles
- Track respawning items with countdown timers and deferred creation
- Serialize/deserialize static objects and switches to game save format
- Cache sprite frames and sounds for pre-loaded static types
- Handle switch state transitions and touch-plate interactions

## External Dependencies
- **Memory**: `z_zone.h` — `Z_LevelMalloc()`, `Z_Free()`
- **Graphics**: `lumpy.h` — sprite/patch structures; `rt_draw.h` — `PreCacheLump()`, `PreCacheGroup()`, `SetLight()` 
- **Audio**: `rt_sound.h` — `SD_PreCacheSound()`, `SD_PlaySoundRTP()`
- **Map/Level**: `rt_ted.h` — wall/tile structures; global `tilemap`, `mapplanes`, `MAPSPOT()`, `actorat`, `spotvis`, `LightsInArea`
- **Entities**: `rt_main.h` — `locplayerstate`, `gamestate`, `MISCVARS`, `new`, `tics`
- **Utilities**: `rt_util.h` — `GameRandomNumber()`, `FindEmptyTile()`, `IsPlatform()`, `PlatformHeight()`
- **Game State**: `rt_net.h`, `rt_menu.h`, `rt_view.h` (battle mode, player info)
- **Actors**: `rt_door.h`, actor spawning (`GetNewActor()`, `NewState()`, `MakeActive()`, `SpawnNewObj()`)

# rott/rt_stat.h
## File Purpose
Header for the static objects (sprites, decorations, items, hazards) subsystem. Defines structures and functions to spawn, manage, animate, and save/load static game entities placed in levels—everything from weapons and powerups to environmental hazards and decorative objects.

## Core Responsibilities
- Define static object type enumeration (`stat_t`) and metadata structures
- Manage linked lists of active/inactive static objects
- Provide spawning and initialization functions for various object types
- Handle state transitions (activate/deactivate) and spatial management
- Implement animated wall updates and sprite frame logic
- Support save/restore of static state and respawn queues
- Manage switches and lighting state

## External Dependencies
- **Includes:** `rt_ted.h` (map structures, wall definitions, spawn locations, lighting)
- **Defined elsewhere:**
  - `thingtype` (entity type enum, likely from a core types header)
  - `fixed` (fixed-point number type, likely from math headers)
  - `byte`, `word`, `signed char` (primitive type defs)
  - `dirtype` (direction enumeration)
  - `MAPSPOT()`, `MAPSIZE`, `NUMAREAS` (map access macros/constants)
  - `VBLCOUNTER`, `MAXPLAYERS` (timing/game constants)
  - Lighting subsystem (called by `ActivateLight`, `TurnOnLight`, etc.)

# rott/rt_state.c
## File Purpose
Defines all finite state machine states for game entities (enemies, hazards, effects, player). Each state specifies animation frame (sprite), duration, behavior function, and transition to next state.

## Core Responsibilities
- Declare static state structures for 50+ entity types
- Define state chains for animation sequences (standing, walking, attacking, dying)
- Link state behavior functions (T_Chase, T_Path, A_Shoot, etc.)
- Organize state transitions for actor state machines
- Support both shareware and full-game entity sets via conditional compilation

## External Dependencies
- **sprites.h**: `SPR_LOWGRD_W41`, `SPR_EXPLOSION1`, etc.—sprite/shape IDs
- **states.h**: `statetype` struct definition, `MAXSTATES` constant, state declarations
- **rt_def.h**: `TILESHIFT`, `TILEGLOBAL`, tile/map definitions
- **rt_actor.h**: Actor behavior functions (`T_Chase`, `A_Shoot`, `T_Projectile`, etc.)
- **Behavior Functions** (defined elsewhere): `T_Stand`, `T_Path`, `T_Chase`, `A_Shoot`, `T_Collide`, `T_Explode`, `T_Projectile`, `ActorMovement`, `T_Roll`, `T_BossDied`, etc.

**Notes:**
- File is 4000+ lines; all content is state declarations with no functional code.
- Conditional `#if (SHAREWARE == 0)` sections exclude boss/special entity states in shareware version.
- States reference function pointers like `T_Player`, `T_NME_Explode`, `T_DarkmonkChase`—these are defined in rt_actor.c or other behavior modules.
- `condition` field (signed char) uses flags like `SF_CLOSE`, `SF_CRUSH`, `SF_SOUND` to tag special state properties.

# rott/rt_str.c
## File Purpose
Implements string rendering, text input, and window management for the ROTT game engine. Provides functions for drawing strings in multiple styles (clipped, proportional, intensity-colored), measuring text, handling interactive text input with cursor feedback, and rendering window frames.

## Core Responsibilities
- String and character drawing with various rendering modes (clipped, proportional, intensity-based)
- Text measurement and font metric calculation
- Interactive text input with cursor blinking and editing (line input and password input)
- Window frame rendering using sprite-based borders
- User-facing text printing with alignment and wrapping
- Low-level VGA text mode output for early boot or debugging
- Intensity-based colored font rendering with embedded formatting codes

## External Dependencies
- **Standard library:** `stdlib.h`, `stdio.h`, `stdarg.h`, `string.h`, `ctype.h`
- **Engine core:** `rt_def.h` (constants, types), `rt_menu.h` (menu structures), `rt_in.h` (keyboard input), `rt_vid.h` (video output)
- **Graphics:** `lumpy.h` (font_t, pic_t structures), `modexlib.h` (VGA primitives)
- **Memory & assets:** `w_wad.h` (W_CacheLumpNum), `z_zone.h` (zone allocator)
- **Utilities & subsystems:** `rt_util.h`, `rt_build.h` (menu buffer), `rt_sound.h` (MN_PlayMenuSnd), `isr.h`, `rt_main.h`, `memcheck.h`
- **Defined elsewhere:** `CurrentFont`, `IFont`, `bufferofs`, `ylookup[]`, `linewidth`, `egacolor[]`, `intensitytable`, `Keyboard[]`, `LastScan`, `ticcount`, `VBLCOUNTER`, `PrintX`, `PrintY`, `WindowX/W/Y/H`, `px`, `py`, `bufferheight`, `bufferwidth`, `MONOPRESENT`

# rott/rt_str.h
## File Purpose
Public header for the RT_STR.C string/text module. Declares functions for string measurement and rendering (including proportional and intensity-based variants), text input handling, window management, and geometric data structures used throughout the game engine's UI and text output systems.

## Core Responsibilities
- String measurement (proportional, intensity, basic) and rendering (clipped, centered, buffered)
- Numeric printing (signed/unsigned integers with custom radix support)
- Text input handling with line editing, defaults, and constraints
- Window drawing and positioning on screen
- Intensity-based font color mapping and rendering
- Definition of geometric primitives (Point, Rect) and screen state (WindowRec)
- Callback registration for custom measure/print routines

## External Dependencies
- **lumpy.h**: Defines graphics structures (font_t, pic_t, patch_t, etc.) used for asset data
- **myprint.h**: Lower-level text rendering (DrawText, TextBox, TextFrame, myprintf) and color constants
- Defined elsewhere: Video/graphics rendering backend (VWB/VW functions), input system, palette management

# rott/rt_swift.c
## File Purpose
Provides SWIFT device abstraction and control for Cyberman 3D input devices in ROTT. Wraps DPMI real-mode interrupt calls and DOS memory management to communicate with SWIFT extensions via the mouse driver (INT 0x33).

## Core Responsibilities
- Initialize/terminate SWIFT device detection and resource management
- Query attached SWIFT device type and static/dynamic capabilities
- Generate tactile feedback output to Cyberman device
- Manage DOS real-mode memory buffers for device communication
- Execute DPMI real-mode interrupts for device I/O

## External Dependencies
- **Notable includes:** `<dos.h>` (DOS interrupt support), `"rt_def.h"` (engine definitions), `"rt_swift.h"` (public API), `"_rt_swft.h"` (private definitions), `"memcheck.h"` (debug memory tracker).
- **External symbols used (defined elsewhere):**
  - `_dos_getvect()`, `int386()`, `int386x()`, `segread()` — DOS/DPMI interrupt functions (from DOS extender runtime).
  - `SoftError()` — debug error logging function (RT engine).
  - `memset()` — standard C library.
  - `allocDOS()`, `freeDOS()` — defined in this file (static); not exported.

# rott/rt_swift.h
## File Purpose
Public header declaring the interface for SWIFT haptic feedback device support. Provides initialization, device detection, input polling, and tactile feedback control for specialized 3D input devices in Rise of the Triad.

## Core Responsibilities
- Initialize and detect presence of SWIFT device extensions
- Terminate and free SWIFT-related resources
- Query attached device type and static configuration
- Poll 6DOF input status (position, orientation, buttons) each frame
- Generate tactile feedback (motor on/off cycling)
- Read dynamic device state and capabilities

## External Dependencies
- **Included:** `rt_playr.h` — provides `SWIFT_3DStatus` and `SWIFT_StaticData` typedef definitions
- **Memory model:** Far pointers indicate real-mode DOS protected mode or segmented memory architecture

# rott/rt_table.h
## File Purpose

Defines the global state lookup table (`statetable`) that maps numeric state IDs to pointers to `statetype` definitions. This file serves as the central registry enabling O(1) runtime access to all entity behaviors, animations, and effect sequences by state ID.

## Core Responsibilities

- Provide a global indexed lookup table for game state definitions
- Initialize 660–1300 state pointers depending on game version (shareware vs full)
- Organize states by entity type: guards, enemies, NPCs, projectiles, effects, environmental
- Support hierarchical state machines by index (e.g., state 0 = s_lowgrdstand, state 1 = s_lowgrdpath4, etc.)
- Maintain symmetry between state ID and pointer position in array via careful ordering

## External Dependencies

- **Include:** `states.h`
  - Defines `statetype` struct (rotate, shapenum, tictime, think function pointer, condition, next)
  - Defines `MAXSTATES` constant (1300 or 660)
  - Declares all individual state objects (e.g., `extern statetype s_lowgrdstand`)
  
- **External state symbols used but defined elsewhere:**  
  All `s_*` identifiers (e.g., `s_lowgrdstand`, `s_explosion1`, `s_player`, `s_darkmonkstand`) are declared in `states.h` and implemented in other `.c` files. This file only collects pointers to them.

# rott/rt_ted.c
## File Purpose
Handles ROTT level/map file loading, precaching of level resources, and comprehensive level initialization including spawning actors, setting up doors, walls, and other interactive elements. Acts as the primary level setup orchestrator.

## Core Responsibilities
- Load map data from ROTT and Ted format files with RLEZ decompression
- Manage precache system for graphics, sounds, and other resources
- Initialize all tile-based entities (walls, doors, switches, animated walls)
- Spawn player start positions and multiplayer team locations
- Configure actors (enemies), static objects (pickups, hazards), and special geometry
- Handle map-specific features (elevators, platforms, lights, masked walls)
- Support version checking, game mode conversions (shareware/registered/low-memory)

## External Dependencies
- **Map I/O & Loading:**
  - `w_wad.h`: `W_CacheLumpNum`, `W_GetNumForName`, `W_LumpLength`
  - `z_zone.h`: `Z_Malloc`, `Z_Heap*`, zone memory manager
  - Standard C I/O: `<stdio.h>`, `<io.h>` (DOS `lseek`, `read`, `close`)
- **Actor/Object Spawning:**
  - `rt_actor.h`, `rt_stat.h`, `rt_door.h`: Spawn functions for all entity types
  - `rt_playr.h`: Player state initialization
- **Audio/Music:**
  - `rt_sound.h`: `SD_PreCacheSound*`, `MU_StartSong` (music system)
- **Graphics/Rendering:**
  - `rt_vid.h`, `rt_draw.h`, `rt_scale.h`: Display functions (`DrawNormalSprite`, `VW_UpdateScreen`)
  - `modexlib.h`, `engine.h`: Low-level graphics primitives
- **Game Systems:**
  - `rt_def.h`: Core constants, macros (MAPSPOT, AREATILE, etc.)
  - `rt_util.h`, `rt_cfg.h`: Utility/config functions
  - `rt_floor.h`, `rt_view.h`, `rt_main.h`: Other game systems
- **Utilities:**
  - `watcom.h`: Fixed-point math macros (`FixedMulShift`)
  - `develop.h`, `rt_debug.h`: Debug/development features

# rott/rt_ted.h
## File Purpose
Header for Ted (level editor) integration, declaring map/level loading, entity setup, and game initialization. Defines data structures for levels, walls, teams, spawn points, and clocks; exports functions to load maps, configure game entities (players, doors, walls, switches, lights), and manage level resources.

## Core Responsibilities
- Define map file formats and in-memory level representations (RTLMAP, mapinfo_t)
- Export map loading and level initialization pipeline
- Manage spawn locations, team assignments, and player setup
- Configure map entities: walls, doors, switches, clocks, lights, animated walls
- Provide asset precaching and map metadata lookup
- Handle special level features (exits, platforms, push walls, links)

## External Dependencies
- **rottnet.h**: Provides `MAXPLAYERS`, `boolean`, `byte`, `word` typedefs; networking context
- **Implied:** thingtype (entity type enum), MAPSPOT macro, NUMAREAS constant, VBLCOUNTER (video sync), various subsystem functions (not declared here)

# rott/rt_text.c
## File Purpose
Implements custom text markup rendering and page layout for the ROTT game engine. Parses a domain-specific markup language (with commands like `^C`, `^G`, `^P`) to render formatted text, graphics, and paginated articles with dynamic margin management.

## Core Responsibilities
- Parse and execute custom text formatting commands (`^C` color, `^G` graphic, `^P` page break, `^L` locate, `^T` timed graphic, `^E` end, `^B` bar)
- Implement word-wrapping text layout with per-row margin constraints
- Manage graphics insertion and adjust text margins around graphics
- Navigate between pages in multi-page articles
- Cache graphics resources before rendering
- Handle keyboard input for page navigation (up/down/escape)

## External Dependencies
- **Graphics/Video:** `W_CacheLumpNum()`, `W_GetNumForName()`, `VW_UpdateScreen()`, `VWB_DrawPic()`, `VWB_Bar()`, `VWB_DrawPropString()`, `VW_MeasurePropString()`, `MenuFadeIn()`
- **Input:** `IN_ClearKeysDown()`, global `LastScan` (keyboard scan code), global `ticcount` (frame counter)
- **Scan codes referenced:** `sc_UpArrow`, `sc_PgUp`, `sc_LeftArrow`, `sc_DownArrow`, `sc_PgDn`, `sc_RightArrow`, `sc_Enter`, `sc_Escape`
- **Standard library:** `stdlib.h` (atoi), `ctype.h` (toupper, isdigit implied), `string.h` (strcpy, strcat)
- **Project headers:** `RT_DEF.H` (types, constants), `memcheck.h` (memory debugging)

# rott/rt_util.c
## File Purpose
Utility module providing low-level services for the ROTT engine: palette/color management, safe file I/O with error handling, memory allocation wrappers, mathematical approximations (distance, angle), path/string parsing, and debug logging.

## Core Responsibilities
- Palette and color lookup (EGA color mapping, RGB-to-palette quantization, gamma correction)
- Safe file operations with chunking for large files (32KB threshold)
- Zone memory allocation wrapper with error checking
- Mathematical approximations (2D/3D distance, arctangent via octant lookup)
- Command-line argument parsing and number format conversion
- Debug/error logging to files (with conditional compilation gates)
- Direct video memory text output and graphics primitives
- DOS-style path/drive navigation
- Generic heap sort with custom comparison/switch callbacks

## External Dependencies
- **System headers:** `stdio.h`, `stdlib.h`, `string.h`, `malloc.h`, `dos.h`, `fcntl.h`, `errno.h`, `io.h`, `ctype.h`, `direct.h`, `sys/stat.h`
- **Game-specific:** `z_zone.h` (Z_Malloc, Z_Free, Z_LevelMalloc, zonememorystarted), `rt_in.h` (IN_UpdateKeyboard, Keyboard array), `rt_vid.h` (VL_ClearVideo, possibly gamma tables), `rt_main.h` (ShutDown, gamestate, player), `rt_dr_a.h`, `rt_playr.h`, `scriplib.h` (GetToken, script variables), `rt_menu.h`, `rt_cfg.h`, `rt_view.h`, `modexlib.h`, `version.h` (ROTT_ERR, version constants)
- **Engine headers:** `watcom.h` (FixedMul, FixedDiv2, FixedMulShift), `develop.h` (development/debug macros)
- **Externals:** `_argc`, `_argv` (command-line), `player` (player object), `gamestate` (game state record), `gammatable`, `gammaindex`, `SOUNDSETUP`

# rott/rt_util.h
## File Purpose
Utility header declaring functions for palette management, file I/O, memory allocation, graphics operations, and hardware-level port I/O. Core support layer for the ROTT game engine providing safe wrappers around system resources and utility helpers for color, math, and file path handling.

## Core Responsibilities
- Palette acquisition, modification, and EGA color mapping
- Safe file I/O operations with error handling
- Memory allocation with level-based tracking
- File path parsing and manipulation
- Graphics screen and buffer operations
- Hardware port I/O for graphics mode setup
- String/number parsing and byte-order conversions
- Math utilities (distance, angle approximation)
- Error reporting and debug output
- Command-line parameter checking

## External Dependencies
- **Local includes:** `develop.h` (build flags: DEBUG, SOFTERROR, TEXTMENUS, SHAREWARE, SUPERROTT, etc.)
- **Implied external symbols:** Standard C file I/O (open, read, write, close), malloc/free, printf/sprintf-style functions, hardware graphics mode setup (VGA port writes)
- **Language features:** `#pragma aux` (Watcom C inline assembly); variadic functions (`...`)

# rott/rt_vh_a.asm
## File Purpose
Low-level x86 assembly module providing video hardware abstraction and joystick input routines for the ROTT engine. Handles tiled screen buffer updates to VGA memory and analog joystick position reading via resistor-capacitor timing.

## Core Responsibilities
- Implement screen tile update logic with VGA register manipulation
- Read joystick analog values using port-based resistor discharge timing
- Manage VGA write planes and graphics controller state during screen updates
- Convert raw joystick timing counts to calibrated position values

## External Dependencies
- **I/O Ports:** 0201h (game control port), 03C4h (VGA sequencer index), 03CEh (VGA graphics controller)
- **External symbols:** _bufferofs, _displayofs, _linewidth, _blockstarts, _update, _Joy_x, _Joy_y, _Joy_xb, _Joy_yb, _Joy_xs, _Joy_ys

# rott/rt_vh_a.h
## File Purpose
Public header declaring interface functions to assembly-language video hardware and input handling code. Declares screen update and joystick input functions with register preservation constraints via compiler pragma.

## Core Responsibilities
- Declare screen update/refresh entry point (`VH_UpdateScreen`)
- Declare joystick input reading entry point (`JoyStick_Vals`)
- Specify register clobbering information for assembly functions via `#pragma aux`

## External Dependencies
- Implementation in `rt_vh_a.asm` (assembly language module)
- Uses Watcom C pragmas (`#pragma aux`) for register-level calling conventions

# rott/rt_vid.c
## File Purpose
Core video/graphics subsystem for VGA Mode-X rendering. Implements double-buffered screen updates via dirty rectangles, drawing primitives (bars, lines, pictures), palette management, fade/transition effects, and LBM image decompression. Directly interfaces with VGA hardware via planar graphics mode.

## Core Responsibilities
- **VGA Mode-X rendering**: Planar graphics memory access with per-plane writes
- **Double-buffering**: Update block tracking to minimize screen transfers
- **Drawing primitives**: Bars, lines (horizontal/vertical/arbitrary), pictures, tiled regions
- **Palette operations**: Fill, set, get color; fade/fade-in/transition effects
- **Image handling**: LBM decompression and screen output
- **Screen management**: Border color control, update flushing

## External Dependencies
- **System includes:** `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<dos.h>`, `<conio.h>` (DOS/Watcom)
- **Key local headers:** 
  - `rt_def.h` (constants, types: UPDATEWIDE, UPDATEHIGH, UPDATESIZE)
  - `_rt_vid.h` (private macros: PIXTOBLOCK, VW_Hlin/Vlin wrappers)
  - `lumpy.h` (pic_t, lbm_t struct definitions)
  - `modexlib.h` (VGA low-level, likely VGAMAPMASK, VGAREADMAP, VGAWRITEMAP, SCREENBWIDE)
  - `rt_view.h` (VH_UpdateScreen, ThreeDRefresh, CalcTics)
  - `rt_util.h` (SafeMalloc, SafeFree, WaitVBL, VL_GetPalette, VL_SetPalette, FixedMul)
  - `w_wad.h` (W_CacheLumpNum)
- **External symbols** (defined elsewhere):
  - `bufferofs`, `displayofs` (screen buffer offsets)
  - `ylookup[]` (Y-to-offset lookup table)
  - `linewidth` (screen width in bytes)
  - `colormap` (color remapping tables for transparency/shading)
  - `maxshade`, `minshade` (shade table extents)
  - `tics` (elapsed game ticks since last frame)

# rott/rt_vid.h
## File Purpose
Public header for the video/graphics rendering subsystem (RT_VID.C). Declares drawing primitives, screen update functions, and palette management for a software-based tile-rendering engine. Uses a dirty-rect update buffer for efficient screen synchronization.

## Core Responsibilities
- **Screen drawing primitives**: Direct pixel/memory-to-screen transfers, rectangles, lines, and picture blitting
- **Tile-based rendering**: Tiled region drawing with offset support
- **Palette management**: Setting/getting colors, fade effects, palette switching, LBM decompression
- **Dirty-rect updates**: Screen block marking and lazy screen refresh
- **Texture/border effects**: Textured lines/bars, border color management, fade-to-color transitions

## External Dependencies
- **lumpy.h**: pic_t, lpic_t, font_t, lbm_t, patch_t, transpatch_t, cfont_t typedefs
- **C primitives**: byte, boolean, int, unsigned, short, char
- **Implied**: Video hardware access (VGA DAC, framebuffer) defined elsewhere

# rott/rt_view.c
## File Purpose
Manages the 3D view rendering pipeline, including projection calculations, lighting systems, color mapping, and screen layout. Handles focal width adjustments, lightning/periodic lighting effects, and illumination management for the ROTT engine's raycasting renderer.

## Core Responsibilities
- View geometry: projection angles, focal width, screen dimensions, scaling factors
- Lighting: shade levels, dynamic area lighting, lightning flashes, periodic light oscillations
- Color management: loading and applying colormaps for player colors and lighting effects
- Screen setup: UI layout (status bars, kills display), viewport positioning and scaling
- Illumination control: temporary light level adjustments for special effects

## External Dependencies
- **w_wad.h**: W_CacheLumpName, W_GetNumForName, W_LumpLength, W_ReadLump (WAD lump loading: "tables", "colormap", "specmaps", "playmaps", "backtile")
- **z_zone.h**: Z_Malloc (memory allocation for lights array)
- **rt_util.h**: SafeMalloc, SafeFree (safe memory wrappers); FixedMul (fixed-point multiply)
- **rt_game.h**: GameRandomNumber (random number generator); MAPSPOT (map tile access); lights, lightsource, fog, gamestate, numareatiles, LightsInArea (external globals for light sourcing and map state)
- **rt_sound.h**: SD_Play3D, SD_PlayPitchedSound, SD_LIGHTNINGSND (sound effects)
- **rt_draw.h / rt_vid.h**: DrawGameString, VW_MeasurePropString, DrawTiledRegion, DrawPlayScreen, ThreeDRefresh, VL_CopyDisplayToHidden, ylookup (rendering/display)
- **modexlib.h**: Mode X graphics primitives (implicitly used via drawing functions)
- **luminance tables**: sintable (sine lookup; defined elsewhere)

# rott/rt_view.h
## File Purpose
Header for the rendering view subsystem. Declares constants, global state, and functions for managing screen setup, camera/focal width parameters, color palettes, gamma correction, and dynamic lighting/illumination levels in a 1990s-era software 3D engine.

## Core Responsibilities
- Screen initialization and view size configuration (MAXVIEWSIZES: 11 configurable sizes)
- Focal width (field-of-view) adjustment
- Gamma table management (8 levels, 64×8 entries)
- Player color palette selection (11 colors) and colormap loading
- Lighting/illumination system: dynamic per-area levels, darkness/shade ranges, and lightning effects
- Status bar display control (kills, health stats, bottom/top bars)
- Per-tile light query functions

## External Dependencies
- **Includes:** `modexlib.h` (VGA ModeX video mode constants and screen buffer management)
- **Implied definitions:** `rt_def.h` (via modexlib), defining types like `byte`, `longword`, `fixed`, `boolean`
- **Defined elsewhere:** All function implementations; colormap resources; gamma/palette data

# rott/sbconfig.c
## File Purpose
Implements SpaceTec IMC Spaceware button configuration parsing and fixed-point arithmetic value warping. Reads button mappings and warp range definitions from a config file, stores them in static globals, and provides query/transformation functions to scale input values through piecewise-linear lookup tables.

## Core Responsibilities
- Parse button and warp range configuration from text file
- Store configuration in file-static globals (button names, warp records)
- Provide query functions to retrieve button names and warp ranges
- Implement compiler-specific fixed-point multiplication (16.16 format)
- Apply piecewise-linear warp transformations to short integer values
- Parse fixed-point and integer literals from config file strings

## External Dependencies
- **Standard C:** `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<ctype.h>`, `<dos.h>`
- **Game/project headers:** `develop.h` (compiler defines), `sbconfig.h` (type definitions), `memcheck.h` (memory debugging)
- **Defined elsewhere:** `strtol`, `stricmp` (non-standard, from RTL), `malloc`, `realloc`, `free`, `fopen`, `fgets`, `fclose`, `strtok`, `strncpy`, `strcpy`, `isspace`
- **Macros from sbconfig.h:** `INT_TO_FIXED`, `FIXED_ADD`, `FIXED_SUB`, `MAX_STRING_LENGTH`

# rott/sbconfig.h
## File Purpose
Configuration header for Sound Blaster button mappings and warp range (input value scaling) settings. Defines the data structures and public API for parsing `.cfg` files and retrieving button/warp configurations at runtime.

## Core Responsibilities
- Define `WarpRange` and `WarpRecord` structures for value mapping/scaling
- Parse configuration files with VERSION, BUTTON, and RANGE entries
- Provide lookup functions for button name mappings (bidirectional)
- Provide lookup functions for named warp range configurations
- Support value warping (scaling) using fixed-point arithmetic
- Define lexical/syntax rules for configuration file format

## External Dependencies
- `fixed` typedef (defined elsewhere, likely a fixed-point type)
- Configuration file format is custom (see syntax diagram at bottom of file)

# rott/scriplib.c
## File Purpose
Script file parser and token extraction library. Loads script files into memory and provides functions to tokenize and iterate through script content, with support for line tracking, lookahead, and comment handling for a configuration/script processing system.

## Core Responsibilities
- Load entire script files from disk into managed memory buffer
- Tokenize script content by extracting whitespace-delimited words
- Track current line number for error reporting
- Skip whitespace and semicolon-prefixed comment lines
- Provide lookahead capability via token pushback (UnGetToken)
- Support both single-token and end-of-line (full line) extraction modes
- Detect end-of-script conditions

## External Dependencies
- **Notable includes**:
  - `rt_def.h` — defines `boolean`, `byte`, game constants
  - `scriplib.h` — declares these five functions; defines MAXTOKEN
  - `rt_util.h` — declares `Error()` (printf-like error reporting)
  - `memcheck.h` — memory debugging wrapper (optional)
  - Platform-specific: `<io.h>`, `<dos.h>`, `<fcntl.h>` for DOS file I/O; `<libc.h>` for NeXTSTEP UNIX

- **Defined elsewhere**:
  - `LoadFile(filename, **bufferptr)` — file I/O utility; returns file size
  - `Error(format, ...)` — error reporting; likely aborts on error
  - `strcpy()` — C standard library

# rott/scriplib.h
## File Purpose
Public header for the script parsing/tokenization subsystem. Declares the interface for loading script files and extracting tokens, supporting line-based text processing with lookahead capability.

## Core Responsibilities
- Define global state for active script buffers and parsing position
- Declare token extraction functions (sequential and with lookahead)
- Track parsing context (line number, end-of-script state)
- Support token availability checking

## External Dependencies
- None visible; header is self-contained declaration only.

# rott/snd_reg.h
## File Purpose
Sound registry and configuration file that maps digital sound identifiers to engine sound parameters, priorities, and mixing flags. This is the primary sound lookup table for the game engine, organizing hundreds of game sound effects across categories like menu, weapons, player actions, enemies, and environmental effects.

## Core Responsibilities
- Define enum of all digital sound IDs used throughout the game
- Map each digital sound to its MUSE (sound subsystem) equivalent
- Assign sound priority levels for mixer arbitration (menu > game > secondary > environmental)
- Configure sound playback flags (pitch shift enabled/disabled, playonce, write, etc.)
- Organize sounds by functional category (menus, weapons, actors, environment, secrets)

## External Dependencies
- **`sound_t`** — struct type containing digital ID, MUSE ID, flags, and priority (defined elsewhere)
- **`MAXSOUNDS`** — array size constant (defined elsewhere)
- **MUSE sound IDs** — e.g., `MUSE_MENUFLIPSND`, `MUSE_LASTSOUND` (defined elsewhere, likely in sound driver header)
- **Sound flag constants** — e.g., `SD_PITCHSHIFTOFF`, `SD_PLAYONCE`, `SD_WRITE` (defined elsewhere)
- **Priority constants** — e.g., `SD_PRIOMENU`, `SD_PRIOPCAUSD`, `SD_PRIOBOSS`, `SD_PRIOREMOTE` (defined elsewhere)


# rott/snd_shar.h
## File Purpose
Sound enumeration and static sound table for the ROTT game engine. Maps ~300+ game sound events to digital audio samples and MUSE music system equivalents, with playback priority and behavior flags.

## Core Responsibilities
- Defines `digisounds` enum with all sound event IDs used throughout the game
- Initializes static `sounds[]` array mapping each sound to digital/MUSE variants and priority
- Organizes sounds by category (menu, weapons, player, enemies, environment, secrets)
- Specifies sound playback priorities to manage audio resource contention
- Documents pitch-shift and play-once behavior for specific sounds

## External Dependencies
- **Undefined here (defined elsewhere):**
  - `sound_t` typedef – likely `snd_shar.c` or header
  - `D_*` constants – digital sound sample IDs
  - `MUSE_*` constants – MUSE music system sound IDs
  - `SD_*` constants – flags (`SD_WRITE`, `SD_PLAYONCE`, `SD_PITCHSHIFTOFF`) and priority levels (`SD_PRIOMENU`, `SD_PRIOGAME`, `SD_PRIOBOSS`, etc.)
  - `MAXSOUNDS` – array size constant


# rott/sndcards.h
## File Purpose
Header file defining enumerated sound card types supported by the game engine. Provides a centralized list of hardware sound devices that can be initialized and used for audio output, along with a version identifier for the audio subsystem.

## Core Responsibilities
- Define enumeration of supported sound cards (SoundBlaster, Adlib, GenMidi, SoundCanvas, etc.)
- Provide version identifier for the audio subsystem
- Serve as interface for audio initialization and device selection elsewhere in codebase

## External Dependencies
- Standard C preprocessor (for `#define`, `#ifndef` guards)
- No explicit includes; used by external modules for type definitions


# rott/splib.h
## File Purpose
Header file defining the SpaceWare input device driver interface for the ROTT engine. Provides data structures, function prototypes, and utility wrappers for communicating with 3D mouse/input devices (specifically the SpaceWare Avenger) through TSR interrupt-based driver calls.

## Core Responsibilities
- Define packet structures for driver/device communication (open, force, button data)
- Define TSR interrupt function codes (driver open/close, device control, data retrieval)
- Define enums for device types and input event classifications
- Provide function prototypes for driver lifecycle (open/close)
- Provide function prototypes for device lifecycle (open/close, enable/disable)
- Provide function prototypes for raw data polling (force vectors, button states)
- Provide convenience wrapper functions that aggregate input into a simplified data structure

## External Dependencies
- **Conditional compiler directives**: `__cplusplus` (C++ extern guard), `_MSC_VER`, `__BORLANDC__` (REALMODE memory model detection)
- **FAR keyword**: Conditionally defined for real-mode vs. protected-mode compilation
- **Documentation reference**: "SpReadme.doc" (external user guide)
- No external library dependencies; purely C header definitions


# rott/sprites.h
## File Purpose
Defines enumerated sprite identifiers for all actors, enemies, effects, and weapons in the ROTT engine. This is a centralized sprite ID registry used by game logic and rendering systems to reference animations and visual assets.

## Core Responsibilities
- Define sprite IDs for enemy types (guards, monks, bosses) with their animation sequences (standing, walking, shooting, pain, death)
- Define sprite IDs for hazards and environmental effects (blades, fire jets, crushers, explosions, gibs)
- Define sprite IDs for power-ups, collectibles, and interactive objects
- Define weapon sprite IDs and animations
- Provide sprite aliases for code reuse (e.g., fallback sprites, shared animations)
- Organize sprite IDs by category with clear naming conventions (prefixes like `SPR_`, `W_`, actor names)

## External Dependencies
- **develop.h**: Provides compilation flags (`SHAREWARE`, `SHAREWARE == 0`) to conditionally include/exclude shareware-restricted sprites


# rott/spw_int.h
## File Purpose
Header file for SpaceTec spacemouse/input device integration. Defines the packet structure used to communicate input data (6DOF motion and button states) from SpaceTec hardware to the game engine.

## Core Responsibilities
- Define the `Spw_IntPacket` structure for spacemouse data
- Declare button ID constants for spacemouse buttons
- Declare initialization, polling, and cleanup functions for the SpaceTec device driver
- Provide a data format contract between the hardware driver (implementation) and consumers

## External Dependencies
- No includes visible (header-only declarations)
- SpaceTec hardware driver implementation (defined elsewhere)

# rott/states.h
## File Purpose
Declares the core state machine infrastructure for the game engine, including the `statetype` structure that represents finite states for all game entities (enemies, NPCs, effects, projectiles, player). Provides a global state table and extern declarations for hundreds of specific state instances used throughout gameplay.

## Core Responsibilities
- Defines `statetype` structure for state machine nodes (sprite, timing, think function, condition)
- Declares global `statetable` array indexing all available states
- Provides state behavior flags (`SF_*`) for conditional state logic
- Declares extern state objects for all enemy types, effects, and environmental hazards
- Segregates shareware vs. full version state counts via conditional compilation

## External Dependencies
- **Local include**: `develop.h` (debug config, `SHAREWARE` flag)
- **Defined elsewhere**: All `statetype` struct definitions; `think` function implementations for each state's AI logic


# rott/task_man.h
## File Purpose
Public header for a low-level timer task scheduler that manages periodic task execution with priority support. Provides interrupt-safe scheduling and dispatching of callback-based tasks at specified rates.

## Core Responsibilities
- Define task structure and scheduler interface
- Register and manage periodic tasks with rate and priority
- Dispatch scheduled tasks based on elapsed time
- Track interrupt context to prevent unsafe concurrent access
- Provide memory locking for interrupt-safe operation
- Handle task lifecycle (schedule, terminate, rate adjustment)

## External Dependencies
- None visible; self-contained interface with no included headers shown

# rott/texture.asm
## File Purpose
Low-level x86-32 assembly implementation of texture-mapped scanline rendering. The `TextureLine_` procedure samples from a source texture bitmap and writes pixels to a destination buffer, advancing through UV coordinates with per-pixel increments for perspective-correct texture mapping.

## Core Responsibilities
- Setup and patch runtime values (du/dv increments, texture/destination addresses) into immediate operands via self-modifying code
- Iterate over pixels in a scanline, computing fixed-point texture coordinates
- Perform texture memory lookup (8-bit paletted texture access)
- Write pixels to destination buffer via per-pixel offset indirection

## External Dependencies
- All symbols are `EXTRN` (defined elsewhere, likely in C).
- No system calls or library invocations.
- Direct memory access only (texture read, framebuffer write).

# rott/texture.c
## File Purpose
Implements texture-mapped rasterization for horizontal (X-dominant) scan segments. The `XDominantFill()` function processes scanline data and drives per-segment texture fill operations by computing perspective-corrected texture coordinate gradients and invoking the low-level texture rasterizer.

## Core Responsibilities
- Compute texture coordinate interpolation parameters (du/dv) along scanlines
- Iterate over active scan segments in each scanline
- Calculate perspective-corrected texture coordinates using fixed-point math
- Configure global texture state and invoke `TextureLine()` for hardware/low-level rasterization
- Manage scanline preprocessing and bounds tracking

## External Dependencies
- **Preprocessing:** `PreprocessScanlines()` (defined elsewhere)
- **Texture math:** `FixedDiv2()`, `FixedMul()` (fixed-point operations, likely macros or inlined)
- **Rasterization:** `TextureLine()` (low-level texture fill, defined elsewhere)
- **Global data:** `Scanline[]`, `_minscanline`, `_maxscanline`, `Xmax`, `_texture`, `BufferPosition`, `display_buffer` (all defined elsewhere; likely in a shared rendering context)

# rott/texture.h
## File Purpose
Public interface for texture rendering in the 3D Realms Rise of the Triad engine. Declares global state for texture coordinates, scaling factors, and buffer pointers, plus the core texture rasterization function.

## Core Responsibilities
- Export texture coordinate and dimension state (u, v, count, du, dv)
- Declare source and destination buffer pointers for texture data
- Provide TextureLine() as the main texture rasterization entry point
- Coordinate texture setup before rendering operations

## External Dependencies
- Type definitions: `int32`, `byte` (defined elsewhere, likely in a common types header)
- No visible includes in this file

# rott/tsr.h
## File Purpose
Header file defining the interface for communicating with TSR (Terminate and Stay Resident) device drivers via DOS interrupts. Defines data structures, command codes, and error codes for a hardware device driver protocol supporting force feedback and button input devices.

## Core Responsibilities
- Define packet structures for TSR driver/device initialization and data exchange
- Define interrupt command codes and flags for driver control operations
- Define error codes for driver operation failures
- Establish protocol constants for packet validation (magic data, timeouts)
- Support private driver configuration (conditionally compiled)

## External Dependencies
- Conditional reference to `SPWERR_TSR` macro (defined elsewhere) used in error code construction
- No standard library includes
- Designed for 16-bit x86 DOS environment (register-level protocol using AX, DX, BX, ES, etc.)

# rott/usrhooks.c
## File Purpose
Provides a thin wrapper layer for memory allocation and deallocation that the game engine library requires. Allows customization of how the engine obtains and releases memory by delegating to the Z_Zone memory manager. Explicitly designed as a public hook point for modification.

## Core Responsibilities
- Allocate memory on behalf of library code and return success/error status
- Deallocate memory safely with NULL-pointer validation
- Serve as abstraction layer between library requests and the Z_Zone manager
- Enable user customization of memory handling behavior

## External Dependencies
- **z_zone.h** — Z_Malloc, Z_Free function declarations; PU_STATIC memory tag constant
- **memcheck.h** — Memory debugging instrumentation (compiles out if NOMEMCHECK defined)

# rott/usrhooks.h
## File Purpose
Public header file defining the memory management hook interface for the ROTT engine. Provides abstraction for memory allocation and deallocation operations that may be restricted or customized by the calling program.

## Core Responsibilities
- Define error codes for user hook operations
- Declare memory allocation hook function prototype
- Declare memory deallocation hook function prototype
- Enable libraries to interface with caller-controlled memory management

## External Dependencies
- Standard C (no external dependencies; declarations only)
- Implementation defined in `USRHOOKS.C` (mentioned in module header)

# rott/version.h
## File Purpose
Defines compile-time version constants for the ROTT game engine. Provides a single source of truth for version numbering that is #included by the build system and game code. The combined version macro allows version comparisons without string parsing.

## Core Responsibilities
- Define major and minor version numbers as preprocessor constants
- Compute a combined version integer from component versions
- Provide centralized version information for conditional compilation or runtime version reporting
- Enable consistent version numbering across the entire codebase

## External Dependencies
- GNU GPL v2 license header only; no code dependencies
- Designed to be #included by other translation units

# rott/vrio.h
## File Purpose
API documentation header for Virtual Reality input device integration in ROTT. Defines interrupt-based communication protocol (INT 0x33) for reading VR controller input and sending haptic feedback to the VR device.

## Core Responsibilities
- Document interrupt handler 0x30 (GetVRInput) for reading VR controller state and mouse input
- Document interrupt handler 0x31 (VRFeedback) for sending haptic feedback to the VR device
- Define button bit positions for 16 different VR controller inputs
- Specify angle normalization convention (0..2047 range, no negative angles)
- Document register conventions for passing input parameters and receiving output

## External Dependencies
None—pure API documentation.

# rott/w_wad.c
## File Purpose
WAD (Where's All the Data) file manager for loading and accessing game lumps. Implements virtual filesystem for multi-lump WAD files and single-file lumps, with demand-loaded caching integrated into the zone memory system. Includes optional data corruption detection via CRC checksums.

## Core Responsibilities
- Load WAD files (multi-lump archives) and individual lumps, parsing headers and directory tables
- Maintain global lump registry with file handle, position, and size metadata
- Provide name-to-index and index-to-name lookup services for lumps
- Implement demand-based caching with zone memory tag integration
- Detect modified WADs via CRC checksum verification
- Support lump read/write operations with bounds checking and I/O error detection

## External Dependencies
- **Standard C:** `<stdio.h>`, `<conio.h>`, `<string.h>`, `<malloc.h>`, `<io.h>`, `<fcntl.h>`, `<sys/stat.h>`
- **Local headers:** `rt_def.h` (constants, types), `rt_util.h` (utility functions), `_w_wad.h` (private types: `lumpinfo_t`, `wadinfo_t`, `filelump_t`), `z_zone.h` (memory manager), `rt_crc.h` (CRC calculation), `rt_main.h`, `isr.h`, `develop.h` (configuration macros).
- **External functions used:** `SafeMalloc()`, `ExtractFileBase()`, `Z_Realloc()`, `Z_Malloc()`, `Z_ChangeTag()`, `CalculateCRC()`, `Error()`, `SoftError()` (defined elsewhere).
- **POSIX I/O:** `open()`, `close()`, `read()`, `write()`, `lseek()`, `fstat()` (conditional NeXT compatibility shims provided).

# rott/w_wad.h
## File Purpose
Public interface for WAD file management in the Rise of the Triad engine. WAD files are archives containing game resources (lumps) such as sprites, textures, maps, and other data. This header declares functions to load, query, and cache lumps from disk.

## Core Responsibilities
- Initialize single or multiple WAD files from disk
- Look up lumps by name or numeric index
- Query lump metadata (length, total count)
- Read lump data into caller-supplied buffers
- Cache lumps in memory with allocation tags for lifetime management

## External Dependencies
None; this is a self-contained interface header with no visible includes or external symbol references.

# rott/watcom.h
## File Purpose
Provides optimized fixed-point arithmetic operations via Watcom C compiler inline assembly pragmas. This utility header enables efficient integer-based fixed-point math (avoiding floating-point overhead) for core game calculations like transformation, scaling, and division on DOS/early hardware platforms.

## Core Responsibilities
- Declare four fixed-point arithmetic functions (multiply, divide, scale)
- Supply Watcom-specific inline assembly implementations using `#pragma aux`
- Abstract hardware-specific fixed-point operations behind portable C function signatures
- Provide rounding and scaling compensation in low-level arithmetic kernels

## External Dependencies
- **Type `fixed`** is defined elsewhere (not in this file).
- **Only active when `__WATCOMC__` is defined;** function bodies are inline assembly pragmas specific to Watcom C. Other compilers would require alternate implementations.

# rott/z_zone.c
## File Purpose
Implements a custom zone-based memory allocator for the ROTT game engine, supporting two separate zones (main and level) with automatic fragmentation management. Provides memory allocation with tagging-based purging levels, allowing lower-priority blocks to be freed when allocation pressure requires.

## Core Responsibilities
- Allocate and deallocate memory blocks from main and level zones
- Track allocated blocks with metadata (size, owner pointer, purge tag)
- Automatically purge purgeable blocks (tag ≥ 100) when allocation fails
- Coalesce adjacent free blocks to reduce fragmentation
- Maintain memory statistics and heap validation for debugging
- Query available contiguous memory via DPMI interrupt
- Support level transitions by bulk-freeing memory in tag ranges

## External Dependencies
- **System headers:** `<stdio.h>`, `<stdlib.h>`, `<dos.h>`, `<string.h>`, `<conio.h>` – DOS/C runtime
- **Internal headers:** `rt_def.h` (constants), `_z_zone.h` (private structures), `z_zone.h` (public interface), `rt_util.h` (Error, SoftError, SafeMalloc, SafeFree, CheckParm)
- **Conditional headers:** `rt_main.h` (if DEVELOPMENT=1), `develop.h` (feature flags), `memcheck.h` (unused)
- **External functions used:** `GamePacketSize()`, `ConsoleIsServer()`, `UL_DisplayMemoryError()`, `int386x()` (DPMI), `Error()`, `SoftError()`, `SafeMalloc()`, `SafeFree()` – defined elsewhere
- **Macros/constants:** `PU_*` tags from z_zone.h; `MINFRAGMENT`, `MAXMEMORYSIZE`, `LEVELZONESIZE` from _z_zone.h; `FP_SEG()`, `FP_OFF()` for far pointers

# rott/z_zone.h
## File Purpose
Public interface for Z_Zone, Carmack's memory manager for the engine. Defines memory allocation tags, lifecycle constants, and memory management function declarations. Implements a tagged-memory system where allocations are freed based on lifetime categories (static, game, level, cache).

## Core Responsibilities
- Define memory tag constants (PU_*) that categorize allocations by lifetime
- Declare memory allocation (`Z_Malloc`, `Z_LevelMalloc`) and deallocation (`Z_Free`, `Z_FreeTags`) functions
- Provide heap query and statistics functions (`Z_HeapSize`, `Z_UsedHeap`, `Z_AvailHeap`)
- Expose heap debugging and integrity checking (`Z_DumpHeap`, `Z_CheckHeap`)
- Manage tag reassignment for dynamic purge behavior (`Z_ChangeTag`)

## External Dependencies
None (pure header; no includes or external references).

