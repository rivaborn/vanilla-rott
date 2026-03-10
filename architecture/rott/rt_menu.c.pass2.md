# rott/rt_menu.c — Enhanced Analysis

## Architectural Role

rt_menu.c is the **central UI modal system** that intercepts gameplay when ESC or function keys are pressed. It manages all user-facing configuration, game state transitions (load/save/new game), battle mode setup, and hardware initialization dialogs. Critically, it acts as a **state machine dispatcher**: menu selections set flags like `playstate` (ex_resetgame, ex_loadedgame) that the main game loop (rt_main.c) polls to transition engine state. The file is never called per-frame; instead, it blocks the main loop entirely while handling user input.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_main.c**: Calls `ControlPanel()` from the main input handler when ESC or F1–F10 is pressed (blocks game execution)
- **rt_game.c**: May check menu-set flags like `loadedgame`, `NewGame`, `ingame` to trigger game transitions
- **rt_net.c**: Accesses menu state flags for multiplayer game setup validation

### Outgoing (what this file depends on)
- **rt_cfg.c**: `SaveTheGame()`, `LoadTheGame()`, `GetSavedHeader()`, `WriteConfig()` — game persistence
- **rt_in.h**: `IN_UpdateKeyboard()`, `IN_ReadControl()`, mouse/joystick input primitives
- **rt_sound.h, fx_man.h, audiolib/**: `SD_Startup()`, `SD_Shutdown()`, `MU_Startup()`, `MU_Shutdown()` — audio device setup (primary reason for 10k+ lines: sound card enumeration UI)
- **rt_draw.h, rt_view.h, rt_vid.h**: `VWB_DrawPic()`, `VL_FadeOut()`, screen primitives for menu rendering
- **rt_battl.c**: `BATTLE_SetOptions()`, `BATTLE_GetOptions()`, battle mode initialization
- **rt_game.h**: Reads/writes `gamestate`, `playstate`, `GamePaused` for state transitions
- **w_wad.h**: `W_CacheLumpName()`, resource loading for graphics/fonts (fonts: tinyfont, smallfont, bigfont)

## Design Patterns & Rationale

1. **Hierarchical Modal Menu Pattern**: `HandleMenu()` is a reusable loop that accepts a menu item array (`CP_itemtype[]`) and layout metadata (`CP_iteminfo`). Sub-menus (Options → Detail → High/Medium/Low) reuse the same loop, simplifying nested navigation.

2. **Unified Input Abstraction** (`ReadAnyControl()`): Converts keyboard, mouse (interrupt-based), and joystick (calibrated delta) into a single `ControlInfo` struct with cardinal directions + buttons. This decouples menu logic from input hardware and enabled support for unusual devices (SpaceBall, CyberMan detected at runtime).

3. **Callback Dispatch**: Menu items are function pointers (e.g., `CP_NewGame`, `CP_LoadGame`); selection triggers the callback. No virtual tables, just direct pointers—simple but tightly coupled.

4. **Cursor Animation via Static Lookup Table**: `CursorFrame[24]` cycles through sprite frames (0→8→0 ping-pong). Avoids per-frame state by pre-computing the animation sequence; cursor advances one frame per menu input.

5. **Static String Lookup Tables**: `BattleModeDescriptions[]`, `HitPointsOptionDescriptions[]`, etc. encode all option text as compile-time constants. This saves RAM and makes localization difficult (would require rebuild).

## Data Flow Through This File

```
User Key Press (ESC/F1-F10)
    ↓
ControlPanel(scancode)
    ├─→ SetUpControlPanel() [save screen buffer]
    ├─→ CP_MainMenu() / CP_F1Help() / etc. [dispatch to appropriate menu]
    │   ├─→ HandleMenu() loop [input polling + cursor animation]
    │   │   ├─→ ReadAnyControl() [unify keyboard/mouse/joy input]
    │   │   ├─→ Callback (e.g., CP_NewGame)
    │   │   │   └─→ Modifies gamestate.battlemode, playstate, StartGame flag
    │   │   └─→ Returns selected item or escape
    │   └─→ Sub-menus (e.g., CP_SoundSetup) → SD_Startup() [hardware init]
    └─→ CleanUpControlPanel() [restore screen buffer, resume game]

Game Loop polls:
    if (playstate == ex_resetgame) → Initialize new game
    if (playstate == ex_loadedgame) → Load from disk
```

Key state variables: `inmenu`, `ingame`, `playstate`, `NewGame`, `quicksaveslot`, `loadedgame`

## Learning Notes

**Idiomatic to this era/engine:**
- **No async UI**: All menu operations block; no background rendering or deferred input processing. This is typical of DOS-era games where polling was the standard.
- **Explicit device initialization**: Sound/input hardware requires explicit init/shutdown; no plug-and-play. The UI itself is partially a hardware configuration wizard.
- **Memory conservation via WAD lumps**: Graphics/fonts loaded from `.wad` files by name lookup (W_CacheLumpName) rather than dynamic allocation. Reduces fragmentation.
- **Direct x86 interrupt calls**: `int386()` for mouse and joystick; no abstraction layer. Reflects hardware-direct programming common in the early 90s.

**Modern engines do this differently:**
- Event-driven input (key/mouse events pumped through event queues) vs. polled input
- Async/non-blocking UI systems (ImGui-style immediate-mode or retained-mode layouts)
- Unified audio abstraction (OpenAL, SDL_mixer) hiding hardware details
- Asset pipelines with hot reloading instead of lump caching

**Cross-cutting insight:**
- The sheer size of rt_menu.c (10k+ lines) reflects the lack of separation between *menu logic* and *hardware initialization logic*. Sound card setup alone occupies hundreds of lines. A modern engine would move hardware init to a separate subsystem (AudioManager, InputManager).

## Potential Issues

1. **Monolithic + Tightly Coupled**: No clear module boundaries within the file. Sub-system UIs (sound setup, control mapping, battle options) are interleaved rather than isolated. Adding a new configuration option requires editing this single file.

2. **Global State Pollution**: Menu functions set flags (`ingame`, `inmenu`, `playstate`, `NewGame`) that other subsystems poll. No explicit state machine API; relies on convention that rt_main.c checks these after `ControlPanel()` returns.

3. **Unsafe Input Validation** (inference): Functions like `LoadTheGame()` are called with user-selected save slots. No visible bounds checking or version validation beyond a call to `GetSavedHeader()`—if corrupted data exists, crashes are possible.

4. **Hardcoded Shareware Restrictions**: `#ifdef` guards and conditional menu items (e.g., certain battle modes disabled in shareware) are scattered through callbacks. Maintainability concern if feature flags proliferate.

5. **Cursor Animation Desync Risk**: `CursorNum` is a global incremented per input event, not per frame. If input latency spikes (slow I/O during save), cursor animation could stutter noticeably.

---

**Note on Architecture Context**: The ARCHITECTURE CONTEXT subsystem overview exceeded context limits and was not available for this analysis. Cross-references inferred from the xref excerpt and first-pass doc only.
