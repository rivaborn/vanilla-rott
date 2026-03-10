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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rottcom_t` | struct | Communication protocol between ROTT and network driver; holds command type, remote node ID, packet data, and session metadata (player count, game type, tic step) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rottcom` | `rottcom_t` (or pointer in Watcom builds) | global extern | Active communication buffer for command passing between game and driver |
| `pause` | `boolean` | global extern | Global pause state flag |

## Key Functions / Methods

### ShutdownROTTCOM
- Signature: `void ShutdownROTTCOM ( void );`
- Purpose: Cleanly shut down network communication subsystem.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Likely releases driver resources, closes communication channel.
- Calls: Not visible in this file (defined elsewhere).
- Notes: Called during game shutdown or when exiting multiplayer session.

### CheckParm
- Signature: `int CheckParm (char *check);`
- Purpose: Scan command-line arguments for a specific parameter.
- Inputs: Parameter string to search for.
- Outputs/Return: Status/index or boolean result (defined elsewhere).
- Side effects: None.
- Calls: Defined in global.h context.
- Notes: Shared utility used across multiple subsystems.

### LaunchROTT
- Signature: `void LaunchROTT (void);`
- Purpose: Spawn the ROTT game executable as a child process.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Creates child process; game state configured via `rottcom` structure prior to call.
- Calls: Likely system-level process creation (not visible).
- Notes: Launcher/stub pattern typical of 1990s multiplayer games; driver runs separately.

### NetISR
- Signature: `void NetISR (void);`
- Purpose: Interrupt service routine for network events.
- Inputs: None (triggered by hardware interrupt via `intnum` field in `rottcom_t`).
- Outputs/Return: None.
- Side effects: Updates `rottcom` structure with incoming packets or completion status.
- Calls: Not visible in this file.
- Notes: Handles asynchronous network I/O; called when `rottcom.intnum` interrupt fires.

### GetVector
- Signature: `long GetVector (void);`
- Purpose: Retrieve the current interrupt vector (likely for the network driver).
- Inputs: None.
- Outputs/Return: Interrupt vector address.
- Side effects: None.
- Calls: Not visible in this file.
- Notes: Used during initialization to hook or query the network interrupt handler.

## Control Flow Notes
This header establishes **inter-process communication (IPC)** between two separate executables:
1. **Launcher** (ROTTCOM driver) initializes, configures `rottcom_t`, and calls `LaunchROTT()`.
2. **Game process** (ROTT.EXE) reads/writes to shared `rottcom` structure and triggers `NetISR` via interrupt.
3. On game exit, `ShutdownROTTCOM()` is called to clean up.

Not a direct part of game init/frame/render loop but rather a communication **boundary layer** that decouples the game from hardware/driver details.

## External Dependencies
- **Conditional includes**: `develop.h` (Watcom builds, debug config) or `global.h` (shareware/retail, shared types).
- **I/O functions**: `outp()`, `inp()` (legacy DOS/x86 port I/O macros, defined in global.h or compiler runtime).
- **Types**: `boolean`, `short`, `char`, `long` (defined in global.h).
- **Watcom-specific**: `#pragma pack` for struct alignment; conditional `rottcom` pointer vs. direct reference depending on compiler.
- **External symbols used but not defined here**: `rottcom` (shared memory region or pointer), `pause` flag, interrupt handler setup, process spawning.
