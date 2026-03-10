# rott/rt_com.h

## File Purpose
Public interface for ROTT networking and packet communication. Declares initialization routines, packet I/O functions, and synchronization utilities for multi-player game sessions.

## Core Responsibilities
- Initialize the network subsystem (`InitROTTNET`)
- Read and write game network packets
- Manage packet buffers and transit timing
- Expose global networking state (player ID, sync clock)

## Key Types / Data Structures
None defined in this file. Refers to `rottcom_t` (defined in rottnet.h).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `badpacket` | int | external | Flag indicating corrupted/invalid packet |
| `consoleplayer` | int | external | ID of local player (0–MAXPLAYERS) |
| `ROTTpacket` | byte[MAXCOMBUFFERSIZE] | external | Buffer for network packet data |
| `controlsynctime` | int | external | Synchronization clock for game ticks |

## Key Functions / Methods

### InitROTTNET
- Signature: `void InitROTTNET(void)`
- Purpose: Initialize the network driver and prepare for packet I/O
- Inputs: None
- Outputs: None
- Side effects: Sets up global network state
- Calls: Not inferable from this file
- Notes: Should be called once during game startup

### ReadPacket
- Signature: `boolean ReadPacket(void)`
- Purpose: Retrieve the next pending network packet
- Inputs: None
- Outputs: Returns true if a packet was read, false if queue empty
- Side effects: Populates `ROTTpacket` buffer; sets `consoleplayer`, `remotenode`
- Calls: Not inferable from this file
- Notes: Non-blocking; checks `remotenode` field in rottcom struct

### WritePacket
- Signature: `void WritePacket(void *buffer, int len, int destination)`
- Purpose: Queue a packet for transmission to a remote node
- Inputs: `buffer` (data to send), `len` (byte count), `destination` (remote node ID)
- Outputs: None
- Side effects: Writes to network outbound queue
- Calls: Not inferable from this file
- Notes: Non-blocking queue operation

### SetTime
- Signature: `void SetTime(void)`
- Purpose: Synchronize local clock with network/game frame timing
- Inputs: None
- Outputs: None
- Side effects: Updates `controlsynctime`
- Calls: Not inferable from this file
- Notes: Likely called once per frame to maintain tick synchronization

### GetTransitTime
- Signature: `int GetTransitTime(int client)`
- Purpose: Query network latency / round-trip time to a specific client
- Inputs: `client` (remote node ID)
- Outputs: Transit time (units not specified; likely milliseconds)
- Side effects: None apparent
- Calls: Not inferable from this file
- Notes: Used for lag compensation or diagnostics

## Control Flow Notes
Part of the game's **network/communication layer**. Called from:
- **Init**: `InitROTTNET` during game startup
- **Main loop**: `ReadPacket` / `WritePacket` each frame to exchange state
- **Sync**: `SetTime` and `GetTransitTime` for tick synchronization

## External Dependencies
- `rottnet.h` — Defines `rottcom_t`, platform macros (`__WATCOMC__`), and max buffer sizes
- Implies a driver/platform layer (DOS/network driver for modem or LAN games)
