# rott/_rt_net.h

## File Purpose
Private header for network packet and command synchronization infrastructure in multiplayer RoTT. Defines macros for accessing command buffers, packet addressing, network timeouts, and declares packet handling functions and status enums.

## Core Responsibilities
- Define macro accessors for player/client/server command buffers indexed by time
- Define packet addressing and command state lookup macros
- Specify network timeout constants (standard, modem, server)
- Declare packet lifecycle functions (prepare, send, receive, resend, process)
- Declare status checking and synchronization functions
- Define enums for setup states, command status, and player presence

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `setupcheckforpacketstate` | enum | States during packet setup verification (nodata, gameready, data, done) |
| `en_CommandStatus` | enum | Command buffer states (ready, notarrived, fixing) |
| `en_playerstatus` | enum | Player presence in game (ingame, quitgame, leftgame) |

## Global / File-Static State
None. (Macros reference external arrays: `PlayerCmds`, `ClientCmds`, `LocalCmds`, `ServerCmds`, `CommandState` defined elsewhere.)

## Key Functions / Methods

### PreparePacket
- Signature: `void PreparePacket(MoveType * pkt)`
- Purpose: Initialize/format a packet for transmission
- Inputs: Packet pointer (MoveType)
- Outputs/Return: None
- Side effects: Modifies packet structure
- Calls: Not inferable from this file

### SendPacket
- Signature: `void SendPacket(void * pkt, int dest)`
- Purpose: Transmit a packet to destination
- Inputs: Packet data, destination identifier
- Outputs/Return: None
- Side effects: Network I/O
- Calls: Not inferable from this file

### GetRemotePacket
- Signature: `void GetRemotePacket(int from, int delay)`
- Purpose: Retrieve/process packet received from remote peer
- Inputs: Source peer, time delay/offset
- Outputs/Return: None
- Side effects: Updates command buffer state
- Calls: Not inferable from this file

### ResendLocalPackets / ResendServerPackets
- Signature: `void ResendLocalPackets(int time, int dest, int numpackets)` / `void ResendServerPackets(int time, int dest, int numpackets)`
- Purpose: Retransmit buffered packets for reliability
- Inputs: Start time, destination, packet count
- Outputs/Return: None
- Side effects: Network retransmission
- Calls: Not inferable from this file

### ProcessPacket / AddServerPacket / AddClientPacket
- Signature: `void ProcessPacket(void * pkt, int src)` / `void AddServerPacket(void * pkt, int src)` / `void AddClientPacket(void * pkt, int src)`
- Purpose: Parse/integrate received packet into command buffer
- Inputs: Packet data, source identifier
- Outputs/Return: None
- Side effects: Updates `CommandState`, command arrays
- Calls: Not inferable from this file

### AllPlayersReady / AreClientsReady / IsServerCommandReady
- Signature: `boolean AllPlayersReady(void)` / `boolean AreClientsReady(void)` / `boolean IsServerCommandReady(int time)`
- Purpose: Check if peers have acknowledged/completed commands at given time
- Inputs: Optional time parameter
- Outputs/Return: Boolean status
- Side effects: None (query only)
- Calls: Not inferable from this file

### RequestPacket
- Signature: `void RequestPacket(int time, int dest, int numpackets)`
- Purpose: Request retransmission of missing packets from peer
- Inputs: Time range, destination, packet count
- Outputs/Return: None
- Side effects: Network request transmission
- Calls: Not inferable from this file

## Control Flow Notes
This header is core multiplayer synchronization infrastructure. Commands are buffered per player/client/server in ring buffers (MAXCMDS-sized, indexed via `CommandAddress` macro). Macros like `NextLocalCommand()` and `NextServerCommand()` step through time-indexed command rings. The `CommandState` array tracks per-command/per-peer acknowledgment status (ready/notarrived/fixing), enabling retransmission on stall. Packet resend and fixup routines handle lossy network conditions. This integrates into the main game loop's control update phase.

## External Dependencies
- **Includes**: None visible (private header structure only)
- **External types**: `MoveType` (packet data), `COM_ServerHeaderType` (server header)
- **External symbols**: 
  - Global arrays: `PlayerCmds`, `ClientCmds`, `LocalCmds`, `ServerCmds`, `CommandState`
  - Timing: `controlupdatestartedtime`, `controlupdatetime`, `serverupdatetime`, `VBLCOUNTER` (vertical blank counter)
  - Constants: `MAXCMDS` (command ring buffer size)
