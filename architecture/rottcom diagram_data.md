# rottcom/rottipx/global.c
## File Purpose
Provides core utility functions for error handling and command-line argument parsing in the IPX networking setup subsystem. The Error function implements abnormal termination with formatted output, while CheckParm enables parameter discovery during initialization.

## Core Responsibilities
- Print formatted error messages with variadic arguments
- Orchestrate clean shutdown before exit
- Locate and return command-line parameter positions
- Support DOS/real-mode runtime environment

## External Dependencies
- **Includes:** `<stdarg.h>` (va_list, va_start, va_end), `<stdlib.h>` (exit), `<stdio.h>` (printf, vprintf), `<string.h>` (stricmp), `<dos.h>` (DOS APIs, likely defunct)
- **Defined elsewhere:** `Shutdown()` (ipxsetup.h), `_argc`/`_argv` (C runtime), `socketid`, `server`, `numnetnodes` (ipxsetup.h externs—not used in this file)

# rottcom/rottipx/global.h
## File Purpose
Global header providing common type definitions, macros, and utility declarations for the Rise of the Triad engine. Establishes platform abstraction layer for I/O operations, boolean types, and system constants.

## Core Responsibilities
- Define portable type aliases (byte, WORD, LONG, boolean)
- Provide hardware I/O abstractions (port input/output, interrupt control) for DOS/x86 compatibility
- Define common constants (TRUE/FALSE, EOS, ESC, clock frequency)
- Declare global utility functions (error reporting, parameter checking)

## External Dependencies
- **Defined elsewhere**: `inp()`, `outp()`, `disable()`, `enable()` — DOS/x86 I/O and interrupt control functions (platform-specific library)
- **No explicit includes shown** — relies on platform headers providing the above functions

# rottcom/rottipx/ipxnet.c
## File Purpose
Implements the IPX (Internetwork Packet Exchange) network driver for the ROTT multiplayer engine. Provides packet allocation, socket management, and send/receive operations via interrupt-driven IPX protocol calls. Handles node discovery and packet sequencing by timestamp.

## Core Responsibilities
- Allocate and deallocate packet buffers dynamically based on server/client role
- Initialize IPX driver detection and socket creation
- Register packet receive buffers with the IPX driver via ECBs (Event Control Blocks)
- Send data packets to remote nodes (unicast or broadcast)
- Receive incoming packets, identify sender, and extract application data
- Maintain a node address lookup table and route packets by node index
- Perform byte-order conversions for network fields

## External Dependencies
- **Includes/imports:** `<dos.h>`, `<process.h>`, `<values.h>` (DOS-specific headers for register access, interrupts); `rottnet.h`, `ipxnet.h`, `ipxsetup.h` (local headers).
- **Extern symbols:** `socketid`, `server`, `numnetnodes` (from `ipxsetup.h`); `rottcom` (global game state struct, used for `.data`, `.datalength`, `.numplayers`, `.remotenode`).
- **System calls:** `geninterrupt(0x2f)` (IPX driver detection via DOS interrupt).
- **Implicit dependency:** IPX driver installed and resident in DOS memory.

# rottcom/rottipx/ipxnet.h
## File Purpose
IPX (Internetwork Packet Exchange) network protocol header defining packet structures, node addressing, and core networking functions for multiplayer communication in the ROTT game engine. Bridges game data with low-level IPX driver primitives.

## Core Responsibilities
- Define IPX packet header format and fields (destination/source network/node/socket)
- Define Event Control Block (ECB) for IPX driver communication
- Provide game-specific payload wrapper (`rottdata_t`) and setup data structure
- Declare network initialization and shutdown functions
- Declare packet send/receive interface functions
- Manage node address registry and time-stamping for packet sequencing

## External Dependencies
- **`c:\merge\rottnet.h`** (absolute path; likely a merged shared header) — defines `MAXPACKETSIZE`, `MAXNETNODES`, and other constants
- **`global.h`** — provides base type definitions (`BYTE`, `WORD`, `boolean`)
- **Defined elsewhere:** `MAXPACKETSIZE`, `MAXNETNODES` constants; implementation of `InitNetwork`, `ShutdownNetwork`, `SendPacket`, `GetPacket`

# rottcom/rottipx/ipxsetup.c
## File Purpose

IPX network setup module for ROTT multiplayer initialization. Orchestrates discovery of network nodes (players), assigns player numbers, and establishes the server/client architecture before launching the game.

## Core Responsibilities

- Parse command-line arguments to determine game mode (server/client, standalone, node count)
- Initialize IPX network layer and configure socket parameters
- Discover available network nodes via broadcast requests and responses
- Assign player numbers and establish client states during synchronization
- Display humorous messages while waiting for players to join
- Handle network address mapping for each node
- Coordinate server/client state transitions and validation

## External Dependencies

**Notable includes / imports:**
- `rottnet.h` – ROTT core networking definitions (`rottcom` struct, `NETWORK_GAME`, packet commands)
- `ipxnet.h` – IPX-specific structures (ECB, IPXPacket, setupdata_t)
- DOS/system headers: `<dos.h>`, `<conio.h>`, `<bios.h>`, `<time.h>`, `<process.h>`

**Defined elsewhere (external symbols):**
- `InitNetwork()`, `ShutdownNetwork()` – IPX driver init/cleanup
- `SendPacket(int dest)`, `GetPacket()` – Network I/O
- `ShutdownROTTCOM()` – ROTT communication cleanup
- `LaunchROTT()` – Start game executable
- `Error(const char *, ...)` – Fatal error handler
- `CheckParm(const char *parm)` – Command-line argument parser
- `rottcom` global struct – Shared communication block with driver
- `nodeadr[]`, `remoteadr`, `remotetime` – Address/timing state managed by ipxnet module
- `randomize()`, `random()` – Standard library random functions
- `bioskey()` – BIOS keyboard input

# rottcom/rottipx/ipxsetup.h
## File Purpose
Header file for IPX network setup and initialization in the ROTT game engine. Declares global network state variables and a shutdown routine for the IPX (Internetwork Packet Exchange) networking layer.

## Core Responsibilities
- Export IPX socket identifier for network communication
- Export server/client mode flag
- Export network node count state
- Declare shutdown procedure for network cleanup

## External Dependencies
- Standard C types (`unsigned`, `int`, `boolean`, `void`)
- Definitions of `boolean` and related types come from elsewhere in the codebase

# rottcom/rottnet.c
## File Purpose
Initializes network communication infrastructure for ROTT by setting up interrupt vectors, allocating a dedicated interrupt stack, and launching the main ROTT executable with network parameters. This is a bridge between the network launcher and the game itself, handling low-level ISR setup required for IPX/serial multiplayer.

## Core Responsibilities
- Allocate and manage a private stack for network interrupt service routine execution
- Find and hook an available DOS interrupt vector (0x60–0x66 range) for network communication
- Save/restore the interrupt vector lifecycle during setup and shutdown
- Parse command-line parameters and build argument list for launching ROTT
- Implement the ROTTNET_ISR interrupt handler wrapper that switches stacks
- Configure ticstep (game tick skipping) based on game type (modem vs. network)
- Spawn the main ROTT executable via `spawnv` and wait for completion

## External Dependencies
- **Standard C:** `<stdio.h>`, `<stdlib.h>`, `<string.h>` (printf, malloc, free, sprintf, sscanf)
- **DOS/Real-mode:** `<process.h>` (spawnv), `<dos.h>` (getvect, setvect, disable/enable, FP_SEG/FP_OFF), `<conio.h>` (getch)
- **Local headers:** 
  - `"rottnet.h"` (defines `rottcom_t`, constants like `ROTTLAUNCHER`)
  - `"global.h"` (Error, CheckParm, boolean, ESC constant)
  - `"port.h"` (conditional on `ROTTSER`; defines `Is8250()`, serial queue structures)
- **Defined elsewhere:** `NetISR()` (called from ROTTNET_ISR; likely in port.c or net.c), `_argc`, `_argv`, `_DS`, `_SS`, `_SP` (compiler/DOS intrinsics)

# rottcom/rottnet.h
## File Purpose
Defines the inter-process communication (IPC) protocol between the ROTT game engine and a separate network/modem driver process. Establishes network configuration constants, declares the shared communication structure, and provides driver control functions.

## Core Responsibilities
- Define `rottcom_t` structure for interrupt-based IPC with network driver
- Establish network multiplayer limits (max 14 nodes, 5–11 players depending on build)
- Declare driver lifecycle functions (launch, shutdown, interrupt handler)
- Support both modem and network game types with configurable player/node/packet parameters
- Provide VGA palette I/O macros for DOS hardware access
- Handle Watcom C compiler-specific structure packing (#pragma pack)

## External Dependencies
- **develop.h** (Watcom) / **global.h** (non-Watcom) — Defines SHAREWARE flag, `boolean` typedef, `outp()` port I/O macro
- **Hardware I/O**: Assumes DOS `outp()` function for VGA palette writes (ports 0x3c8, 0x3c9)
- **Interrupt system**: Assumes ability to read/hook interrupt vectors via GetVector() and ISR setup
- **DOS process model**: Assumes ability to spawn child processes and manage TSR drivers


# rottcom/rottser/global.c
## File Purpose

Utility library providing safe wrappers for file I/O, memory allocation, and error handling for the ROTT multiplayer server setup system. Includes configuration file parameter reading/writing and string manipulation helpers.

## Core Responsibilities

- Error reporting with variadic formatting and program termination
- Command-line argument parsing
- Safe file operations (open, read, write) with I/O error checking and chunking
- Error-checked memory allocation
- Configuration parameter serialization/deserialization from script format
- String length calculation and bounded string copying

## External Dependencies

- **Standard C library:** `malloc.h`, `fcntl.h`, `errno.h`, `stdlib.h`, `stdio.h`, `string.h`, `stdarg.h`, `sys/stat.h`
- **DOS/Windows legacy:** `conio.h`, `io.h`, `dos.h`, `dir.h` (indicates 16-bit DOS/Windows code)
- **Local headers:** `sersetup.h` (calls `ShutDown()`), `scriplib.h` (uses `GetToken()`, `token`, `endofscript`)
- **External symbols:** `_argc`, `_argv` (C runtime), `strerror()`, `open()`, `read()`, `write()`, `close()`, `filelength()`, `strlen()`, `strcpy()`, `itoa()`, `atoi()`, `stricmp()`, `strcmpi()`

# rottcom/rottser/global.h
## File Purpose
Global header for the ROTT serialization/server component. Provides fundamental constants, type definitions, and declarations for utility functions handling error reporting, file I/O, memory management, and string operations.

## Core Responsibilities
- Define basic constants (TRUE, FALSE, clock frequency, ESC character)
- Define low-level I/O and interrupt control macros (INPUT, OUTPUT, CLI, STI)
- Define core data types (boolean, byte, word, longword, fixed-point)
- Declare error handling and fatal exit functions
- Declare file I/O utilities with error handling
- Declare safe memory allocation and string manipulation functions
- Declare command-line parameter parsing utilities

## External Dependencies
- Low-level I/O macros wrap functions defined elsewhere: `inp()`, `outp()`, `disable()`, `enable()` (DOS/x86-specific port I/O and interrupt control)
- No explicit #includes visible; likely includes standard C library headers elsewhere

# rottcom/rottser/keyb.h
## File Purpose
Header file defining keyboard scan code constants for the Rise of the Triad engine. Maps physical keyboard keys (at AT/PS2 hardware level) to named symbolic constants used throughout input event handling.

## Core Responsibilities
- Define symbolic constants for all supported keyboard scan codes
- Provide named identifiers for input event processing (e.g., `sc_Return`, `sc_Escape`)
- Map special keys (function keys, arrows, modifiers, navigation)
- Map alphanumeric keys (A–Z, 0–9)
- Supply fallback/sentinel values (`sc_None`, `sc_Bad`)

## External Dependencies
- None (self-contained constants file)

---

**Notes:**
- All macros use the `sc_` prefix (likely "scan code")
- Scan codes match standard IBM AT/PS2 keyboard protocol
- `sc_Enter` is aliased to `sc_Return` (0x1c)
- `sc_Bad` (0xff) and `sc_None` (0x00) serve as error/no-key sentinel values
- No conditional compilation or platform-specific branching visible

# rottcom/rottser/port.c
## File Purpose
Low-level serial port driver for DOS-based system that manages UART (8250/16550) hardware initialization, interrupt-driven buffered I/O, and modem control signals. Provides serial communication backend for multi-player game networking.

## Core Responsibilities
- Detect UART type (8250 vs 16550) and IRQ via BIOS system data
- Initialize UART with baud rate, divisor, and control line settings (DTR, RTS)
- Hook IRQ vector and manage interrupt service routine
- Maintain circular input/output queues for buffered byte transfer
- Handle UART interrupt events (RX, TX, modem status, line errors)
- Provide high-level read/write API with buffer overflow protection

## External Dependencies
- **Included headers:** `<conio.h>`, `<dos.h>` (x86/DOS primitives), `<stdio.h>`, `<stdlib.h>`, `<mem.h>`
- **Local headers:** `global.h` (CLOCK_FREQUENCY, INPUT/OUTPUT/CLI/STI macros), `port.h` (que_t, extern queues and port config), `serial.h` (UART register offsets and constants), `sercom.h` (statistics externs), `sersetup.h` (shutdown extern)
- **Defined elsewhere:** `getvect()`, `setvect()` (DOS vector table), `inp()`, `outp()`, `int86x()` (DOS I/O), `disable()`, `enable()` (CPU interrupt control), `memcpy()` (std lib); statistics counters (`numBreak`, `numFramingError`, etc.), `writeBufferOverruns` in sercom module

# rottcom/rottser/port.h
## File Purpose
Defines the serial port communication interface for DOS-era hardware, including circular queue data structures for buffered I/O and interrupt service routine declarations for 8250/16550 UART chips. Enables non-blocking serial communication via hardware interrupts.

## Core Responsibilities
- Define circular queue structure (`que_t`) for input/output buffering with power-of-2 sizing
- Declare global configuration variables (IRQ, UART type, COM port, baud rate)
- Declare interrupt service routines (ISRs) for 8250 and 16550 UART chip types
- Declare initialization/shutdown functions for serial port hardware setup
- Provide byte-level and buffer-level read/write functions for queued serial I/O

## External Dependencies
- `<conio.h>`, `<dos.h>` — DOS/BIOS console and system interrupt support (legacy, requires DOS or DPMI mode)
- Hardware: 8250 or 16550 UART chip accessible via I/O port addresses
- Macro `QueSpot(index)` implements power-of-2 circular queue modulo using bitwise AND

# rottcom/rottser/scriplib.c
## File Purpose
Script file tokenizer and parser for the ROTT game engine. Provides utilities to load configuration/script files and extract tokens while tracking line numbers and managing parsing state.

## Core Responsibilities
- Load script files into memory and initialize parser state
- Tokenize input by extracting whitespace-delimited words
- Support both standard tokens and end-of-line tokens (rest of line)
- Skip comments (semicolon-delimited) and whitespace
- Track parse position, line numbers, and EOF state
- Implement one-token lookahead via UnGetToken

## External Dependencies
- **`LoadFile`** (declared in `global.h`, defined elsewhere) – loads file into memory
- **`Error`** (declared in `global.h`, defined elsewhere) – reports parsing errors
- Platform headers: `io.h`, `dos.h`, `fcntl.h` (DOS/NeXT conditional includes)

# rottcom/rottser/scriplib.h
## File Purpose
Public header for a script parsing library. Declares global variables and functions for loading and tokenizing script files in a token-stream model. This is a foundational utility for configuration or scripting systems within the engine.

## Core Responsibilities
- Expose global script parsing state (buffer pointers, current token, line tracking)
- Provide script file loading and initialization
- Declare token acquisition and lookahead functions
- Track EOF and token-availability state

## External Dependencies
- `#include "global.h"` — for `boolean` typedef and utility functions (`SafeMalloc`, file I/O)
- Symbols defined elsewhere: `boolean`, `Error()`, file I/O primitives

# rottcom/rottser/sercom.c
## File Purpose
Serial packet-based communication layer for multiplayer ROTT games. Implements frame-delimited packet I/O, connection negotiation between two players, and comprehensive statistics collection on serial link performance.

## Core Responsibilities
- **Packet I/O**: Read and write variable-length packets with frame character escaping over serial queue
- **Connection negotiation**: Synchronize two game clients via handshake protocol ("ROTT" packets with stage tracking)
- **Network ISR dispatch**: Route CMD_SEND and CMD_GET commands to packet handlers
- **Statistics tracking**: Count bytes, packets, buffer overruns, and UART errors; compute aggregate metrics
- **Timing**: Track game session start/end and elapsed playtime
- **State management**: Maintain escape state machine for packet framing and oversize packet detection

## External Dependencies
- **port.h**: `read_byte()`, `write_buffer()`, `inque` (input queue), `outque` (output queue), `QUESIZE`, `MAXPACKETSIZE`
- **rottnet.h**: `rottcom` (global network command/data structure), `CMD_SEND`, `CMD_GET`
- **sersetup.h**: `showstats`, `usemodem` (globals referenced elsewhere, not used in sercom.c)
- **Standard C (DOS)**: `<time.h>` (time_t, time(), ctime()), `<conio.h>` (bioskey, clrscr), `<stdio.h>` (printf), `<string.h>` (memcpy, strncmp, sprintf, strlen), `<dos.h>` via port.h
- **Defined elsewhere (global.h):** `CheckParm()`, `Error()`, `delay()`, `gettime()`

# rottcom/rottser/sercom.h
## File Purpose
Header declaring the serial communication interface for network gameplay in Rise of the Triad. Provides functions for packet transmission/reception, connection management, interrupt handling, and communication diagnostics.

## Core Responsibilities
- Declare packet-based network I/O functions (send/receive)
- Track serial line error statistics (break, framing, parity, overrun)
- Count interrupt events (Tx/Rx)
- Provide connection setup and interrupt service routine
- Offer timing measurement for network latency analysis
- Reset and report communication counters for debugging

## External Dependencies
- Standard C (`void`, `int`, `char`, `unsigned long`, `boolean`)
- Serial hardware interrupt registration (implied by `NetISR`)
- Timing subsystem (implied by `StartTime`/`EndTime`)
- All implementations defined elsewhere

# rottcom/rottser/serial.h
## File Purpose
Header file defining UART (serial port) register addresses and bit-field constants for direct hardware control. Provides memory-mapped I/O register definitions and bit masks for configuring and communicating with serial devices via a standard 16550-compatible UART interface.

## Core Responsibilities
- Define UART register offsets (addresses) for I/O operations
- Define bit masks and flag constants for interrupt enable/status control
- Define line control register flags (word length, parity, stop bits)
- Define modem control and status register flags (handshake signals)
- Define FIFO control flags for buffer management
- Provide divisor latch constants for baud rate configuration

## External Dependencies
- Standard C preprocessor (`#define`)
- Assumes 16550 UART hardware interface (standard ISA/serial port)
- No external includes or symbol dependencies


# rottcom/rottser/sermodem.c
## File Purpose

Implements modem control and AT command interface for establishing serial multiplayer connections. Handles modem initialization, dialing outbound numbers, answering incoming calls, and disconnecting via DTR control and AT command sequences.

## Core Responsibilities

- Send AT commands to modem with character-level pacing and delays
- Parse and validate modem responses (OK, RING, CONNECT)
- Initialize modem with stored initialization string
- Dial outbound connections using pulse or tone dialing
- Answer incoming calls and await connection establishment
- Disconnect by toggling DTR and issuing hangup command

## External Dependencies

- **System headers:** `<time.h>`, `<stdio.h>`, `<string.h>`, `<bios.h>` (DOS era)
- **Local headers:** global.h, serial.h, port.h, sermodem.h, sersetup.h
- **Defined elsewhere:**
  - `INPUT()`, `OUTPUT()` macros (UART I/O via inp/outp)
  - `uart`, `MODEM_CONTROL_REGISTER`, `MCR_DTR` constants
  - `delay()` function
  - `read_byte()`, `write_buffer()` (serial drivers)
  - `bioskey()` (DOS BIOS keyboard)
  - `usemodem` flag

# rottcom/rottser/sermodem.h
## File Purpose
Header file declaring the modem communication interface for the ROTT network subsystem. Provides function prototypes and external configuration variables for Hayes-compatible modem control (initialization, dialing, answering, hangup).

## Core Responsibilities
- Declare modem command transmission and response reception functions
- Declare modem initialization, dial, and answer control functions
- Declare modem hangup function
- Export modem configuration strings (init, dial, hangup) and mode parameters

## External Dependencies
- `global.h` – defines `boolean`, `char`, and port I/O macros (`INPUT`, `OUTPUT`)

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

# rottcom/rottser/sersetup.h
## File Purpose
Header file declaring the interface for serial game setup and shutdown in ROTT's multiplayer/modem system. Exposes global state flags and entry point functions for initializing networked games.

## Core Responsibilities
- Declare external variables controlling modem and statistics display modes
- Export setup and shutdown functions for serial/networked game sessions
- Provide minimal interface to multiplayer game initialization subsystem

## External Dependencies
- **Includes**: `global.h` (provides `boolean` typedef, utility function declarations)
- **External symbols**: Uses `boolean` type defined in `global.h`; actual implementation of `ShutDown()` and `SetupSerialGame()` defined elsewhere (likely in `sersetup.c`)

# rottcom/rottser/st_cfg.c
## File Purpose
Loads and parses serial/modem configuration from SETUP.ROT and ROTT.ROT script files during game initialization. Extracts modem parameters (init/hangup strings, baud rate, COM port, IRQ, UART) and phone number, populating global variables used by the serial communication layer.

## Core Responsibilities
- Resolve configuration file paths using environment variables
- Load and parse script-based configuration files (SETUP.ROT, ROTT.ROT)
- Validate expected parameter tokens in configuration
- Extract modem initialization and hangup strings with fallback defaults
- Parse serial port hardware settings (baud rate, COM port, IRQ, UART address)
- Read phone number/dial string for modem connections
- Handle "not configured" markers (~) and provide sensible defaults

## External Dependencies
- **Local headers:** global.h, scriplib.h, port.h, sersetup.h, sermodem.h
- **Standard C / DOS:** stdio.h, string.h, stdlib.h, dos.h, io.h, fcntl.h, ctype.h, dir.h, process.h, errno.h, sys/stat.h
- **Defined elsewhere:** `Error()`, `LoadScriptFile()`, `GetToken()`, `GetTokenEOL()` (scriplib module), `access()` (libc), external globals `token[]`, `name[]`, `scriptbuffer` (scriplib), `initstring[]`, `dialstring[]`, `hangupstring[]`, `pulse` (sermodem), `comport`, `irq`, `uart`, `baudrate` (port), `usemodem` (sersetup)

# rottcom/rottser/st_cfg.h
## File Purpose

A header file declaring the setup configuration reading interface for the ROTT game engine. Provides a single entry point to load configuration data during initialization.

## Core Responsibilities

- Declare the `ReadSetup()` function for configuration initialization
- Define the public interface for setup/configuration module

## External Dependencies

- None visible (no includes)
- Implementation file must be located in `rottcom/rottser/` or linked module

# rottcom/rt_net.h
## File Purpose
Defines network packet types and structures for multiplayer game communication. Provides type definitions for all network messages (movement, game state, chat, synchronization) and utility functions to calculate packet sizes for serialization/deserialization.

## Core Responsibilities
- Define network command type constants (COM_DELTA, COM_TEXT, COM_SYNC, etc.)
- Provide packet structure definitions for each message type
- Calculate packet sizes dynamically based on packet type
- Handle server multi-packet aggregation
- Define audio transmission and synchronization check structures

## External Dependencies
- **Includes:** `rottnet.h` (defines MAXPLAYERS, MAXNETNODES, MAXCODENAMELENGTH, rottcom_t structure)
- **Types used:** `gametype`, `specials`, `battle_type`, `word`, `byte`, `boolean` (defined elsewhere)
- **Constants:** `DUMMYPACKETSIZE`, `MAXCODENAMELENGTH`, `COM_SOUND_BUFFERSIZE` (256), `COM_MAXTEXTSTRINGLENGTH` (33)
- **Notes:** Conditional COM_SYNCCHECK define gated on SYNCCHECK macro; allows optional sync validation

