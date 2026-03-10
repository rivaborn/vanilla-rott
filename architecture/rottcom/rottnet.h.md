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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| rottcom_t | struct | Shared communication buffer between ROTT and network driver; encodes command type (send/receive), packet data, remote node ID, player count, game type, and driver configuration |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| rottcom | rottcom_t or rottcom_t* | global (external) | Shared IPC structure; direct in non-Watcom builds, pointer in Watcom (driver-side) |
| pause | boolean | global (external) | Game pause flag accessible to driver |

## Key Functions / Methods

### ShutdownROTTCOM
- Signature: `void ShutdownROTTCOM(void);`
- Purpose: Gracefully shut down network communication and restore system state
- Inputs: None
- Outputs/Return: None
- Side effects: Likely restores interrupt vectors, deallocates resources
- Calls: Not visible in this file
- Notes: Only declared; implementation in separate driver/network module

### LaunchROTT
- Signature: `void LaunchROTT(void);`
- Purpose: Execute ROTT.EXE after network driver is resident
- Inputs: None
- Outputs/Return: None
- Side effects: Spawns child process; may modify environment variables or DOS memory
- Calls: Not visible in this file
- Notes: Typically called by launcher stub after driver initialization

### NetISR
- Signature: `void NetISR(void);`
- Purpose: Interrupt service routine for asynchronous network packet processing
- Inputs: None (reads from interrupt context)
- Outputs/Return: None
- Side effects: Reads/writes `rottcom` structure, may disable interrupts
- Calls: Not visible in this file
- Notes: Hooked to interrupt vector; services driver-to-game communication

### GetVector
- Signature: `long GetVector(void);`
- Purpose: Retrieve interrupt vector used for network ISR
- Inputs: None
- Outputs/Return: Interrupt vector address (long)
- Side effects: Reads CPU interrupt table
- Calls: Not visible in this file
- Notes: Used to save/restore ISR hook points during driver lifecycle

## Control Flow Notes
This header defines the contract between the main ROTT executable and a separate DOS driver (TSR or spawned process). During initialization, the driver becomes resident and hooks an interrupt vector. During gameplay, the game writes commands to `rottcom` (with `remotenode`, `command`, `datalength`, `data`), and the interrupt fires asynchronously to service the request. On shutdown, `ShutdownROTTCOM()` restores the interrupt vector and cleans up. The conditional Watcom/non-Watcom declarations suggest the driver uses Watcom C (with pointer to struct), while the main game uses a non-Watcom compiler (direct struct).

## External Dependencies
- **develop.h** (Watcom) / **global.h** (non-Watcom) — Defines SHAREWARE flag, `boolean` typedef, `outp()` port I/O macro
- **Hardware I/O**: Assumes DOS `outp()` function for VGA palette writes (ports 0x3c8, 0x3c9)
- **Interrupt system**: Assumes ability to read/hook interrupt vectors via GetVector() and ISR setup
- **DOS process model**: Assumes ability to spawn child processes and manage TSR drivers

## Notes
- **Trivial helpers summarized**: `CheckParm()` (re-declared from global.h) is a command-line parameter parser; included for driver/launcher convenience.
- **Shareware vs. full build**: MAXPLAYERS is 5 or 11 depending on SHAREWARE flag.
- **Packet size**: 2048 bytes; designed for modem and IPX network transports.
