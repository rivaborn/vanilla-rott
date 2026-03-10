# rottcom/rottser/sersetup.c

## File Purpose
Entry point for ROTT's serial multiplayer device driver. Initializes serial/modem hardware, parses command-line options (dial/answer/serial mode), facilitates pre-game player communication via "talk mode", and coordinates game launch. Acts as the bridge between the host system's serial layer and the game engine.

## Core Responsibilities
- Parse command-line arguments to determine connection mode (modem dial, modem answer, or direct serial)
- Initialize serial port hardware and UART configuration
- Establish connection (modem handshake or direct serial link)
- Provide interactive "talk mode" for players to communicate before gameplay
- Launch the main ROTT game engine and track game timing
- Coordinate graceful shutdown of all serial/modem subsystems
- Optionally collect and display connection statistics

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `usemodem` | `boolean` | static | Tracks whether modem mode is active (vs. direct serial) |
| `showstats` | `boolean` | static | Tracks whether connection statistics should be displayed on exit |

## Key Functions / Methods

### main
- Signature: `void main(void)`
- Purpose: Initialize the serial/modem subsystem, parse runtime options, manage the connection and game lifecycle
- Inputs: Command-line arguments (checked via `CheckParm()`)
- Outputs: Exits with code 0
- Side effects: Sets global `rottcom` structure fields (numplayers, gametype, consoleplayer), modifies `usemodem`, `showstats`, `pause` flags; initializes UART, port, and modem hardware; launches game engine
- Calls: `clrscr()`, `printf()`, `CheckParm()`, `ReadSetup()`, `GetUart()`, `InitPort()`, `read_byte()` (flush loop), `Dial()`, `Answer()`, `Connect()`, `reset_counters()`, `talk()`, `StartTime()`, `LaunchROTT()`, `EndTime()`, `stats()`, `ShutDown()`, `exit()`
- Notes: Recognizes `-answer`, `-dial`, `-pause`, `-stats` flags. Always sets `rottcom.numplayers=2` and `gametype=MODEM_GAME`. Flushes UART buffer before connection. Game launch is conditional on successful `Connect()`.

### ShutDown
- Signature: `void ShutDown(void)`
- Purpose: Gracefully shut down all serial/modem subsystems before exit
- Inputs: None
- Outputs: None
- Side effects: Hangs up modem (if active), shuts down ROTTCOM layer, shuts down port/UART
- Calls: `hangup_modem()`, `ShutdownROTTCOM()`, `ShutdownPort()`
- Notes: Conditional hangup based on `usemodem` flag

### talk
- Signature: `void talk(void)`
- Purpose: Interactive "talk mode" allowing players to communicate via keyboard/modem before game starts
- Inputs: Keyboard input via `bioskey()`, incoming serial bytes via `read_byte()`
- Outputs: Console output via `printf()`
- Side effects: Reads from BIOS keyboard buffer and serial input queue; writes to serial output buffer and console
- Calls: `clrscr()`, `printf()`, `bioskey()`, `read_byte()`, `write_buffer()`
- Notes: Infinite loop until ESC (0x1B) is pressed. Converts carriage return (0x0D) to line feed (0x0A) before transmission. Flushes both input sources on exit. Displays both local keystrokes and remote input to console.

## Control Flow Notes
**Initialization → Connection → Game → Shutdown:**
1. **Startup**: Parse command-line flags, configure `rottcom` structure, display mode banner
2. **Hardware Init**: `ReadSetup()` → `GetUart()` → `InitPort()` → flush UART
3. **Connection**: If modem mode, call `Dial()` (dial-out) or `Answer()` (auto-answer); else direct serial
4. **Pre-game**: If `Connect()` succeeds, enter `talk()` loop (players communicate until ESC), then `StartTime()`
5. **Game Execution**: `LaunchROTT()` runs the main game engine
6. **Post-game**: `EndTime()` stops timer; optionally display `stats()` if `-stats` flag set
7. **Cleanup**: `ShutDown()` closes modem/port, then `exit(0)`

## External Dependencies
**Includes:**
- `global.h` — type definitions (`boolean`, `byte`, `word`), utility macros (TRUE, FALSE, ESC), error/logging functions
- `sermodem.h` — modem control: `Dial()`, `Answer()`, `hangup_modem()`, modem config strings
- `sercom.h` — communication layer: `Connect()`, `reset_counters()`, `stats()`, `StartTime()`, `EndTime()`, `LaunchROTT()`
- `port.h` — UART/serial I/O: `InitPort()`, `ShutdownPort()`, `GetUart()`, `read_byte()`, `write_buffer()`
- `rottnet.h` — network config (via `..rottnet.h`)
- `st_cfg.h` — setup configuration
- Standard C/DOS: `conio.h`, `stdio.h`, `stdlib.h`, `string.h`, `ctype.h`, `time.h`, `bios.h`, `process.h`, `stdarg.h`

**Defined elsewhere:**
- `CheckParm()` — command-line argument parser
- `ReadSetup()` — load config from file
- `rottcom` — global network config struct
- `pause` — global flag (defined elsewhere)
