# rott/rt_menu.h — Enhanced Analysis

## Architectural Role
This is the **UI dispatch hub** of the engine—the primary interface through which players transition between game states (main menu → new game/load/battle setup/quit). Beyond rendering, it orchestrates complex state changes: initiating network multiplayer flows, managing save/load state, configuring audio, and controlling game initialization. It's a bridge between the input subsystem (rt_in.h) and the bulk of engine systems (graphics, audio, battle, network).

## Key Cross-References

### Incoming (who depends on this file)
- **rt_main.c** (inferred) – Calls `SetUpControlPanel()`, `ControlPanel()` during main game loop
- **Engine entry points** – Call `CP_MainMenu()` to dispatch menu selections → newgame/load/battle/quit
- **Gameplay loop** – Checks `inmenu` global to pause/unpause game; polls `scancode` for quick-save/load hotkeys

### Outgoing (what this file depends on)
- **rt_in.h** – Input polling (`ReadAnyControl`, `IN_ReadControl`); keyboard scanning
- **lumpy.h** – Graphics primitives (fonts: `font_t`, colored font: `cfont_t`; pic resources)
- **Audio subsystem** (inferred) – `MN_PlayMenuSnd()` plays UI feedback
- **Game state** – Modifies `NewGame`, `loadedgame`, `quicksaveslot`, multiplayer config globals
- **Save/load backend** (inferred in rt_save.c) – `CP_LoadGame()` and `QuickSaveGame()` call file I/O
- **Battle/network subsystem** – Battle setup functions configure multiplayer modes

## Design Patterns & Rationale

**Callback Dispatch Pattern:**  
Menu items (`CP_itemtype`) contain `routine` function pointers—each menu selection calls its associated callback. This decouples menu navigation from action logic (open-/closed-principle).

**Generic Pagination Handler:**  
`HandleMultiPageCustomMenu()` is a reusable multi-page list UI with custom redraw and item-selection callbacks. This avoids code duplication across save/load/level-select menus.

**Screen State Capture:**  
`AllocateSavedScreenPtr()` / `FreeSavedScreenPtr()` preserve the game scene behind menus—a common 1990s technique to avoid full screen redraws. Paired with `inmenu` global to pause the game loop.

**No Separate Menu Loop:**  
Unlike modern engines, menus are driven by `ControlPanel(byte scancode)` called from the main game loop—menu state is implicit (managed by callbacks), not explicit. This makes menu code flow less obvious but tightly integrated with game timing.

## Data Flow Through This File

1. **Initialization:** `SetUpControlPanel()` allocates font/screen resources; `DrawMainMenu()` renders initial menu.
2. **Per-Frame Input:** Main loop calls `ControlPanel(scancode)` → scans menu hotkeys, routes input to active menu handler.
3. **User Selection:** Menu item callback (e.g., `CP_LoadGame()`, `CP_PlayerSelection()`) executes → modifies globals (e.g., `loadedgame`, player config) or calls next menu.
4. **State Transition:** When user selects "New Game," `NewGame = true` is set; main loop exits menu, calls game initialization.
5. **Shutdown:** `CleanUpControlPanel()` deallocates resources before level load or exit.

## Learning Notes

**Idiomatic to 1990s DOS engines:**
- No object-oriented state—all state is global or passed via callback context.
- Menu loop is *integrated* into main loop, not separate—unusual by modern standards (where menus run in isolated loops).
- Screen restoration via off-screen buffer (memory-efficient for 320×200 VGA).

**Modern equivalents:**
- Callback pointers → event systems or command patterns.
- Global state (`inmenu`, `NewGame`) → scene/state manager singletons.
- Separate menu loop → UI framework (Dear ImGui, etc.).

**Game engine concepts:**
- This file is the **UI subsystem's public API**—analogous to a scene manager in modern engines, but less abstracted.
- No data-driven UI (no scripts/config files for menu layout)—all menu structure is hard-coded in rt_menu.c.

## Potential Issues

- **Global state explosion:** 30+ extern globals make it hard to reason about side effects—refactoring would require a menu context struct.
- **Tight coupling:** Menu callbacks directly manipulate game state and call subsystems; no clear separation of concerns.
- **No input abstraction:** Scancode-based input is tied to keyboard; joystick/mouse handled elsewhere, increasing fragmentation.
