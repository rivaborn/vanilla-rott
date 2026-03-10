# rott/rt_cfg.h

## File Purpose
Public configuration header for the runtime settings management module. Declares all global configuration variables for audio, input devices, graphics, and gameplay settings, along with functions to persist these settings to disk. Also defines structures for alternate resource loading (sounds, graphics, levels) and combat macros.

## Core Responsibilities
- Declare and expose global configuration variables (audio modes, volumes, input device settings, graphics quality, player preferences)
- Define data structures for alternate resource information and macro commands
- Provide public interface for reading/writing configuration files, scores, and battle configuration
- Handle password/difficulty/character selection persistence and conversion
- Support sound file and vendor management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| AlternateInformation | struct | Specifies path, availability flag, and filename for alternate resource files (sounds, graphics, levels) |
| MacroList | struct | Stores a combat macro: availability flag and null-terminated command string (max 32 chars) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| FXMode, MusicMode | int | global | Audio output device/mode selection |
| MUvolume, FXvolume | int | global | Master and effects volume levels |
| mouseenabled, joystickenabled, joypadenabled | boolean | global | Input device enabled flags |
| joystickport, mouseadjustment, threshold | int | global | Input device configuration (port, sensitivity) |
| NumVoices, NumChannels, NumBits, MidiAddress | int | global | Audio hardware configuration |
| AutoDetailOn, DetailLevel, DoubleClickSpeed, etc. | int/boolean | global | Graphics and UI behavior settings |
| DefaultDifficulty, DefaultPlayerCharacter, DefaultPlayerColor | int | global | Game difficulty and character appearance defaults |
| CodeName | char[9] | global | Player name/identifier |
| ApogeePath | char* | global | Installation/data directory path |
| passwordstring | byte[20] | global | Encoded difficulty/cheat password |
| RemoteSounds, PlayerGraphics, GameLevels, BattleLevels | AlternateInformation | global | Alternate resource file specifications |
| CommbatMacros | MacroList[10] | global | Combat macro commands (max 10) |

## Key Functions / Methods

### ReadConfig
- Signature: `void ReadConfig(void)`
- Purpose: Load all configuration settings from disk file
- Inputs: None (reads from file system)
- Outputs/Return: None (populates extern global variables)
- Side effects: Reads configuration file; populates all global config variables
- Calls: Likely `ReadInt()`, `ReadBoolean()`, file I/O functions
- Notes: Called during initialization

### WriteConfig
- Signature: `void WriteConfig(void)`
- Purpose: Persist all current configuration settings to disk
- Inputs: None (reads from global variables)
- Outputs/Return: None (writes to file system)
- Side effects: Writes configuration file
- Calls: Likely `WriteParameter()`, file I/O functions

### ReadScores / WriteScores
- Purpose: Load and persist player score/statistics data
- Inputs/Outputs: None (read/write from globals and file system)
- Side effects: File I/O

### WriteBattleConfig
- Purpose: Write battle-specific configuration settings
- Side effects: File I/O

### Password Conversion Functions
- `ConvertStringToPasswordString()`, `ConvertPasswordStringToPassword()`, `ConvertPasswordStringToString()`, `ConvertPasswordToPasswordString()`
- Purpose: Bidirectional encoding/decoding of cheat/difficulty passwords
- Inputs: char* or none (read from `passwordstring` global)
- Outputs: None (write to globals or output parameter)

### Helper Functions
- `ReadInt()`, `ReadBoolean()`: Parse and assign config values
- `WriteParameter()`: Write single int parameter to config file
- `ReadSETUPFiles()`: Load initial setup configuration
- `CheckVendor()`: Validate vendor information
- `DeleteSoundFile()`: Remove alternate sound file

## Control Flow Notes
Purely declarative header. Actual implementation in `RT_CFG.C` integrates into startup (load config → apply settings) and shutdown (save config) sequences. Password conversion functions enable difficulty/cheat code persistence.

## External Dependencies
- Standard C types: `int`, `char`, `byte`, `boolean` (boolean defined elsewhere, likely in common header)
- File I/O: assumed in implementation
- Game-specific paths and resource management
