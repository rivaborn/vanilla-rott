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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CustomCtrls | struct | Holds allowed input device configurations for control binding |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| ScanNames | static byte*[] | static | Keyboard scan code to character name mapping (128 entries) |
| ExtScanCodes | static byte[] | static | Scan codes requiring multi-character names (e.g., "F1", "Esc") |
| ExtScanNames | static byte*[] | static | Names for extended scan codes (function keys, modifiers, navigation) |

## Key Functions / Methods

### Menu Entry Points (Main Handlers)
- **DoMainMenu()** – Main menu event loop entry point
- **CP_NewGame()** – Initiates new game flow (difficulty selection, player count)
- **CP_Control()** – Opens control/input configuration menu
- **CP_Sound(), CP_Music(), CP_FX()** – Audio settings handlers
- **CP_Keyboard(), CP_Mouse(), CP_Joystick(), CP_Special()** – Input device configuration
- **CP_Quit()** – Game exit handler
- **CP_SaveGame()** – Save game selection/confirmation

### Menu Drawing Functions
Multiple `Draw*` functions render specific menu screens:
- **DrawCtlScreen(), DrawSoundMenu(), DrawCustomScreen()** – Primary menu panels
- **DrawLoadSaveScreen(), DrawLoadSaveScreenAlt()** – Save/load file UI
- **DrawKeyboardMenu(), DrawCustomJoystick(), DrawCustomMouse()** – Input binding UI
- **DrawDetailMenu(), DrawBattleMenu(), DrawViolenceLevelMenu()** – Gameplay options
- **DrawBattleModes(), DrawSpawnControlMenu()** – Multiplayer battle settings

### Input Binding Functions
- **DefineKey(), DefineMouseBtn(), DefineJoyBtn()** – Interactive key/button binding
- **DefineMouseBtns1/2(), DefineKeyBtns1/2(), DefineJoyBtns1/2()** – Device-specific binding setup
- **DefineKeyMove1/2()** – Keyboard movement binding

### Utility/Settings Functions
- **MouseSensitivity(), DoThreshold()** – Input device calibration
- **SliderMenu()** – Generic numeric parameter adjustment UI (used for volumes, speeds, settings)
- **CP_DisplayMsg()** – Display on-screen message/confirmation dialog
- **IN_GetScanName()** – Look up keyboard scan code name (from rt_in.h)

## Control Flow Notes
This is a menu system header called during the main menu/pause state. The entry point is `DoMainMenu()`, which dispatches to various `CP_*()` handlers based on user selection. Each handler may call `Draw*()` functions to render its UI, then loop on input until user confirms or cancels. No explicit update/frame/render cycle visible here—menu logic is event-driven via input polling.

## External Dependencies
- **rt_in.h** – Input system types: `ScanCode`, `ControlType`, `KeyboardDef`, `JoystickDef`, mouse/joystick hardware state globals
- Undefined external types: `boolean`, `byte`, `word` (standard C typedef aliases, likely from a common header)
- Rendering functions called but not declared: implied to exist in menu implementation (RT_MENU.C)

## Notes
- Heavy use of magic constants for screen positioning and colors; no abstraction layer for resolution-independence
- Keyboard scan code tables are DOS-era PC hardware specific (PC keyboard scan codes 0x00–0x7F)
- Multiple menu drawing/handler pairs (`Draw*` + `CP_*`) suggest a consistent pattern for each menu type
- `QUICKSAVEBACKUP` define and `NUMSAVEGAMES` constant indicate save system integration
