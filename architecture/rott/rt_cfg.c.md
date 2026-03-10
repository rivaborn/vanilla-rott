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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `fx_blaster_config` | struct | Sound Blaster hardware settings (port, IRQ, DMA, MIDI address) |
| `AlternateInformation` | struct | Path and file info for remote sounds/levels |
| `MacroList` | struct | Network combat macro text and availability flag |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SoundName` | char[13] | static | Filename "SOUND.ROT" |
| `ConfigName` | char* | static | Filename "CONFIG.ROT" |
| `ScoresName` | char* | static | Filename "SCORES.ROT" |
| `BattleName` | char* | static | Filename "BATTLE.ROT" |
| `ROTT` | char* | static | Filename "ROTT.ROT" |
| `CONFIG` | char* | static | Filename "SETUP.ROT" |
| `RemoteSounds` | AlternateInformation | global | Alternate sound file path/availability |
| `GameLevels` | AlternateInformation | global | Alternate game level path/availability |
| `BattleLevels` | AlternateInformation | global | Alternate battle level path/availability |
| `CodeName` | char[MAXCODENAMELENGTH] | global | Network multiplayer code name |
| `CommbatMacros` | MacroList[MAXMACROS] | global | Combat macro text strings for network play |
| `FXMode`, `MusicMode` | int | global | Sound device mode IDs |
| `MUvolume`, `FXvolume` | int | global | Music and effects volume (0–255) |
| `SBSettings` | fx_blaster_config | global | Sound Blaster hardware configuration |
| `mouseenabled`, `joystickenabled`, etc. | boolean/int | global | Input device flags and settings |
| `NumVoices`, `NumChannels`, `NumBits` | int | global | Sound sample parameters |
| `MidiAddress` | int | global | MIDI port address (e.g., 0x330) |
| `DefaultDifficulty`, `DefaultPlayerCharacter`, `DefaultPlayerColor` | int | global | Player defaults |
| `passwordstring` | byte[20] | global | Encrypted password bytes |
| `ConfigLoaded` | boolean | global | Flag tracking if config has been loaded |

## Key Functions / Methods

### ReadConfig
- **Signature:** `void ReadConfig(void)`
- **Purpose:** Entry point for loading all configuration files at startup. Loads SOUND.ROT, then CONFIG.ROT and BATTLE.ROT if available.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Loads scriptbuffer via `LoadScriptFile()`, populates global sound/input/game settings, sets `ConfigLoaded=true`, writes error and exits if SOUND.ROT missing
- **Calls:** `SetSoundDefaultValues()`, `SetConfigDefaultValues()`, `SetBattleDefaultValues()`, `LoadScriptFile()`, `ParseSoundFile()`, `ParseConfigFile()`, `ParseBattleFile()`, `ReadScores()`, `Z_Free()`
- **Notes:** Called early in initialization. Applies defaults first, then overlays file values. Files may be deleted if parse fails.

### WriteConfig
- **Signature:** `void WriteConfig(void)`
- **Purpose:** Save all configuration to disk files with human-readable comments. Guards against recursive calls via `inconfig` flag.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Writes SOUND.ROT, CONFIG.ROT, BATTLE.ROT, and SCORES.ROT; opens files with `O_RDWR | O_TEXT | O_CREAT | O_TRUNC`
- **Calls:** `WriteSoundConfig()`, `WriteBattleConfig()`, `WriteScores()`, `WriteParameter()`, `WriteParameterHex()`, `SafeWriteString()`, `ConvertPasswordStringToString()`, `GetPathFromEnvironment()`
- **Notes:** Skips if `!ConfigLoaded` or if reentrancy check `inconfig > 0`. Adds extensive comments documenting valid values for each option.

### ParseConfigFile
- **Signature:** `boolean ParseConfigFile(void)`
- **Purpose:** Parse CONFIG.ROT script, extracting mouse/joystick/keyboard settings, visual options, and key bindings.
- **Inputs:** None (reads from global `scriptbuffer` populated by `LoadScriptFile()`)
- **Outputs/Return:** `true` if version matches `ROTTVERSION`, `false` otherwise
- **Side effects:** Updates global variables for input settings, view size, detail level, all 27 keyboard button scans, mouse/joystick button mappings, joystick calibration bounds; calls `IN_SetupJoy()` if joystick enabled and calibrated
- **Calls:** `ReadInt()`, `ReadBoolean()`, `ReadUnsigned()`, `GetToken()`, `TokenAvailable()`, `ConvertStringToPasswordString()`, `IN_SetupJoy()`, `unlink()`
- **Notes:** Validates hardware presence (mouse, joystick, Cyberman, Assassin, SpaceBall) and disables devices if not detected. Ignores joystick if calibration values are zero.

### ParseSoundFile
- **Signature:** `boolean ParseSoundFile(void)`
- **Purpose:** Parse SOUND.ROT script for audio hardware configuration.
- **Inputs:** None
- **Outputs/Return:** `true` if version matches, `false` otherwise
- **Side effects:** Updates `MusicMode`, `FXMode`, `MUvolume`, `FXvolume`, `NumVoices`, `NumChannels`, `NumBits`, `MidiAddress`, `stereoreversed`, and `SBSettings` fields
- **Calls:** `ReadInt()`, `ReadBoolean()`, `ReadUnsigned()`
- **Notes:** Reads Sound Blaster hardware parameters (Type, Address, Interrupt, DMA8/16, MIDI, Emu).

### ParseBattleFile
- **Signature:** `boolean ParseBattleFile(void)`
- **Purpose:** Parse BATTLE.ROT script for battle mode configuration (gravity, weapon ammo, spawn settings, time limits, powerup durations).
- **Inputs:** None
- **Outputs/Return:** `true` if version matches, `false` otherwise
- **Side effects:** Updates `BATTLE_Options[]` array for each battle mode (battle_Normal through battle_NumBattleModes), updates `BattleSpecialsTimes` struct, updates `BATTLE_ShowKillCount` and `battlegibs`
- **Calls:** `ReadInt()`, `ReadBoolean()`
- **Notes:** Complex conditional logic — different battle modes skip certain fields (e.g., battle_Eluder skips ammo, weapons, health). Validates Speed/Ammo/LightLevel ranges and uses defaults if out of bounds.

### WriteSoundConfig
- **Signature:** `void WriteSoundConfig(void)`
- **Purpose:** Write SOUND.ROT file with formatted sound device settings and extensive documentation.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Creates/truncates SOUND.ROT; writes all global sound variables with enumerated option descriptions
- **Calls:** `WriteParameter()`, `WriteParameterHex()`, `SafeWriteString()`, `GetPathFromEnvironment()`, `open()`, `close()`
- **Notes:** Skips entirely if `!WriteSoundFile` flag is false. File includes comments explaining Music Modes (0=Off, 1=UltraSound, 2=SoundBlaster, etc.) and FX Modes.

### WriteBattleConfig
- **Signature:** `void WriteBattleConfig(void)`
- **Purpose:** Write BATTLE.ROT file with battle mode rules for each mode and extensive documentation of all options.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Creates/truncates BATTLE.ROT; iterates all battle modes and writes per-mode settings with conditional skips
- **Calls:** `WriteParameter()`, `SafeWriteString()`, `GetPathFromEnvironment()`, `open()`, `close()`
- **Notes:** Very long function (~500 lines). Generates extensive comment sections documenting gravity, speed, ammo, hitpoints, spawn options, friendly fire, light levels, point goals, danger damage, time limits, respawn times.

### ReadSETUPFiles
- **Signature:** `void ReadSETUPFiles(void)`
- **Purpose:** Parse SETUP.ROT and ROTT.ROT to locate alternate sound/level files and extract network settings (code name, combat macros).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Populates `RemoteSounds`, `GameLevels`, `BattleLevels` AlternateInformation structs; sets `CodeName` and `CommbatMacros[]`; deletes ROTT.ROT after parsing
- **Calls:** `GetPathFromEnvironment()`, `LoadScriptFile()`, `GetTokenEOL()`, `GetAlternatePath()`, `GetAlternateFile()`, `GetToken()`, `Z_Free()`, `unlink()`
- **Notes:** SETUP.ROT is main setup file; ROTT.ROT is override file (deleted after use). Skips alternate files in shareware version.

### ConvertPasswordToPasswordString
- **Signature:** `void ConvertPasswordToPasswordString(void)`
- **Purpose:** Encrypt `pword[]` (12 bytes) and `gamestate.violence` (1 byte) into `passwordstring[]` using XOR cipher with hex key.
- **Inputs:** None (reads global `pword` array and `gamestate.violence`)
- **Outputs/Return:** None
- **Side effects:** Updates global `passwordstring[13]`
- **Calls:** None
- **Notes:** Uses fixed key `PASSWORDENCRYPTER = "7d7e4a2d3b6a0319554654231f6d2a"`. XOR is reversible so encryption/decryption use same logic.

### ConvertPasswordStringToPassword
- **Signature:** `void ConvertPasswordStringToPassword(void)`
- **Purpose:** Decrypt `passwordstring[13]` back to `pword[]` and `gamestate.violence` using XOR cipher.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates global `pword[12]` and `gamestate.violence`
- **Calls:** None
- **Notes:** Inverse of `ConvertPasswordToPasswordString()`. Validates decrypted violence level (0–3) and zeroes if out of range.

### ConvertStringToPasswordString
- **Signature:** `void ConvertStringToPasswordString(char * string)`
- **Purpose:** Parse 26-character hex string into 13 bytes of `passwordstring[]`.
- **Inputs:** `string` - pointer to 26-char hex string (interpreted as pairs of hex digits)
- **Outputs/Return:** None
- **Side effects:** Updates global `passwordstring[13]`
- **Calls:** `memset()`, `memcpy()`, `sscanf()`
- **Notes:** Used during config file parsing; reads from CONFIG.ROT "SecretPassword" token.

### ConvertPasswordStringToString
- **Signature:** `void ConvertPasswordStringToString(char * string)`
- **Purpose:** Convert 13 bytes of `passwordstring[]` to 26-character hex string for file output.
- **Inputs:** `string` - output buffer (must hold 26+ bytes)
- **Outputs/Return:** None
- **Side effects:** Writes to output buffer
- **Calls:** `itoa()`, `memset()`
- **Notes:** Each byte becomes two hex digits (high nybble, low nybble).

### SetConfigDefaultValues
- **Signature:** `void SetConfigDefaultValues(void)`
- **Purpose:** Initialize all game configuration globals to factory defaults.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets mouse enabled, joystick disabled, viewsize=7, gamma=0, violence=3, default password bytes
- **Calls:** None
- **Notes:** Called before loading CONFIG.ROT so missing file leaves sensible defaults.

### SetSoundDefaultValues
- **Signature:** `void SetSoundDefaultValues(void)`
- **Purpose:** Initialize sound configuration to defaults and query FX library for detected Sound Blaster settings.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `MusicMode=0`, `FXMode=0`, `NumVoices=4`, `NumChannels=1`, `NumBits=8`, `MidiAddress=0x330`, `stereoreversed=false`; calls `FX_GetBlasterSettings()` and copies detected settings to global `SBSettings`
- **Calls:** `FX_GetBlasterSettings()`
- **Notes:** Allows hardware autodetection to populate Sound Blaster config if available.

### SetBattleDefaultValues
- **Signature:** `void SetBattleDefaultValues(void)`
- **Purpose:** Initialize all battle modes to factory defaults and apply mode-specific overrides.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Loops through all battle modes, sets gravity, speed, ammo, hitpoints, spawning, powerup flags; applies special rules (e.g., CaptureTheTriad kills=1, Hunter time=1 min)
- **Calls:** None
- **Notes:** Called before loading BATTLE.ROT. Handles 9 different battle modes with per-mode tuning.

### CheckVendor
- **Signature:** `void CheckVendor(void)`
- **Purpose:** Verify vendor documentation file (LICENSE.DOC or VENDOR.DOC) integrity against embedded WAD lump via CRC-32 comparison.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** If CRC mismatch or file missing, extracts vendor lump from WAD and writes to disk
- **Calls:** `GetPathFromEnvironment()`, `access()`, `LoadFile()`, `CalculateCRC()`, `SafeFree()`, `W_GetNumForName()`, `W_CacheLumpNum()`, `W_LumpLength()`, `SaveFile()`
- **Notes:** Used to ensure players have required legal documentation. Different lump names for shareware vs. full version.

### ReadScores
- **Signature:** `void ReadScores(void)`
- **Purpose:** Load high-score table from SCORES.ROT file if present.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Reads binary `Scores` struct from file; if file absent, sets `gamestate.violence=0`
- **Calls:** `GetPathFromEnvironment()`, `access()`, `SafeOpenRead()`, `SafeRead()`, `close()`
- **Notes:** Only available if `_ROTT_` is defined (main game, not setup tool).

### WriteScores
- **Signature:** `void WriteScores(void)`
- **Purpose:** Save high-score table to SCORES.ROT file.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Creates/truncates SCORES.ROT with binary Scores struct
- **Calls:** `GetPathFromEnvironment()`, `SafeOpenWrite()`, `SafeWrite()`, `close()`
- **Notes:** Only available if `_ROTT_` defined.

### DeleteSoundFile
- **Signature:** `void DeleteSoundFile(void)`
- **Purpose:** Delete SOUND.ROT file.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Unlinks SOUND.ROT from filesystem
- **Calls:** `GetPathFromEnvironment()`, `unlink()`
- **Notes:** Used if parsing SOUND.ROT fails, allowing user to run sound setup again.

### ReadInt, ReadBoolean, ReadUnsigned
- **Signature:** `void ReadInt(const char * s1, int * val)`, etc.
- **Purpose:** Helper functions to parse config script tokens. Match token against label, extract next numeric token, cast to target type.
- **Inputs:** `s1` - label to match; `val` - pointer to variable to update
- **Outputs/Return:** None
- **Side effects:** Updates target variable if token matches
- **Calls:** `GetToken()`, `TokenAvailable()`, `ParseNum()`, `strcmpi()`/`stricmp()`
- **Notes:** `ReadBoolean` and `ReadUnsigned` convert via temporary `int`; no-op if token doesn't match.

### WriteParameter, WriteParameterHex
- **Signature:** `void WriteParameter(int file, const char * s1, int val)`, etc.
- **Purpose:** Helper functions to write config parameters to file in human-readable format.
- **Inputs:** `file` - open file descriptor; `s1` - label; `val` - numeric value
- **Outputs/Return:** None
- **Side effects:** Writes formatted line to file (e.g., "Label  123\n" or "Label  $1F\n")
- **Calls:** `SafeWriteString()`, `itoa()`, `strcpy()`
- **Notes:** `WriteParameterHex` formats value in base-16 with "$" prefix.

## Control Flow Notes
**Initialization phase:** `ReadConfig()` is called early (likely from `main()` or game startup) to load SOUND.ROT, CONFIG.ROT, BATTLE.ROT, and SCORES.ROT. Defaults are applied first, then file values override. `ReadSETUPFiles()` is called to populate alternate content paths and network macros.

**Runtime:** Game state and config variables are read/modified during gameplay (e.g., `gamestate.violence`, input bindings, detail levels).

**Shutdown phase:** `WriteConfig()` is called at exit or on save-settings request to persist all user modifications.

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
