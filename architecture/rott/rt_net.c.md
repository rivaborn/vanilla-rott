# rott/rt_net.c

## File Purpose
Implements network command synchronization, packet routing, and game state management for ROTT's multiplayer mode. Handles the critical game loop for both client and server sides, including control polling, packet buffering, loss recovery, and demo recording/playback.

## Core Responsibilities
- **Command management**: Allocate/deallocate per-player command queues; track packet arrival status
- **Client-side control**: Poll input, advance control time, format and send movement packets
- **Server-side aggregation**: Collect client packets, broadcast aggregated server packet
- **Packet routing**: Dispatch incoming packets by type to appropriate handlers
- **Reliability**: Resend lost packets via COM_REQUEST/COM_FIXUP protocol
- **Synchronization**: Time-sync clients, verify game state consistency (optional SYNCCHECK)
- **Demo system**: Record/playback gameplay for automated testing or demonstration
- **Player state application**: Unpack network commands into actor momentum and buttons

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `CommandType` | struct | Holds MAXCMDS command buffers for one player |
| `CommandStatusType` | struct | Tracks status (ready/notarrived/fixing) for each command |
| `MoveType` | struct | Movement/delta packet: type, time, momentum, angle, buttons, optional sound |
| `NullMoveType` | struct | Null movement (no change) packet |
| `COM_ServerHeaderType` | struct | Server broadcasts: aggregates multiple clients' commands |
| `COM_TextType` | struct | Chat message packet |
| `COM_RequestType` | struct | Client requests resend of lost packet(s) |
| `COM_FixupType` | struct | Resent packets for packet loss recovery |
| `COM_SyncType` | struct | Time synchronization packet |
| `COM_CheckSyncType` | struct | Game state verification (player position, RNG, angle) |
| `COM_GameMasterType` | struct | Game config broadcast (level, difficulty, RNG seed, players) |
| `DemoType` | struct | Demo frame: time, input, buttons |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `LocalCmds` | CommandType* | global | Current player's command queue |
| `ServerCmds` | CommandType* | global | Server's aggregated commands |
| `PlayerCmds[MAXPLAYERS]` | CommandType* | static | Per-player command queues (all players) |
| `ClientCmds[MAXPLAYERS]` | CommandType* | static | Server-side per-client queues |
| `CommandState[MAXPLAYERS+1]` | CommandStatusType* | static | Status tracking for all commands |
| `controlupdatetime` | int | global | Current control/packet timestamp |
| `serverupdatetime` | int | global | Server's current time |
| `demorecord`, `demoplayback` | boolean | static | Demo mode flags |
| `demobuffer`, `demoptr` | byte* | static | Demo file buffer and position |
| `PlayerStatus[MAXPLAYERS]` | enum | static | Per-player (ingame/quit/left) state |
| `GamePaused` | boolean | global | Game pause flag |
| `IsServer`, `modemgame`, `networkgame` | boolean | global | Network mode flags |
| `LastCommandTime[MAXPLAYERS]` | int | static | Last packet time from each player for loss detection |

## Key Functions / Methods

### InitializeGameCommands
- **Signature:** `void InitializeGameCommands(void)`
- **Purpose:** Allocate command buffers, determine server/client role, set control divisor
- **Inputs:** Global flags (modemgame, networkgame, rottcom, consoleplayer, numplayers)
- **Outputs/Return:** None
- **Side effects:** Allocates PlayerCmds, LocalCmds, ServerCmds, ClientCmds; sets IsServer, standalone
- **Calls:** SafeMalloc, SafeLevelMalloc, GamePacketSize, GetTypeSize, memset
- **Notes:** Guard via GameCommandsStarted; server always enables remoteridicule

### UpdateClientControls
- **Signature:** `void UpdateClientControls(void)`
- **Purpose:** Poll input, create packets per control time, send via network
- **Inputs:** Global input (controlbuf, buttonbits), timer (ticcount)
- **Outputs/Return:** None
- **Side effects:** Increments controlupdatetime, dispatches packets via PrepareLocalPacket
- **Calls:** NextLocalCommand, PrepareLocalPacket, UpdateDemoPlayback, CheckForPacket, ProcessServer
- **Notes:** Re-entrant guard via InUCC; creates MoveType or NullMoveType; wraps sound if ready

### ProcessPacket
- **Signature:** `void ProcessPacket(void * pkt, int src)`
- **Purpose:** Dispatch packet by type field to appropriate handler
- **Inputs:** pkt (untyped buffer), src (source player)
- **Outputs/Return:** None
- **Side effects:** Routes to AddPacket, AddServerPacket, ResendPacket, FixupPacket, etc.
- **Calls:** Handlers per type (15+ cases)
- **Notes:** Central dispatcher; handles delta, text, pause, quit, sync, server aggregates

### ProcessServer
- **Signature:** `void ProcessServer(void)`
- **Purpose:** Main server loop: wait for all clients ready, broadcast aggregated packet
- **Inputs:** Global state (IsServer, UpdateServer, serverupdatetime, controlupdatetime)
- **Outputs/Return:** None
- **Side effects:** Blocks until AreClientsReady, calls SendFullServerPacket, respects timeouts
- **Calls:** AreClientsReady, CheckForPacket, UpdateClientControls, SendFullServerPacket
- **Notes:** Guard via InProcessServer; exits on restartgame flag; handles timeouts

### ControlPlayerObj
- **Signature:** `void ControlPlayerObj(objtype * ob)`
- **Purpose:** Apply queued command to player object (momentum, buttons)
- **Inputs:** ob (player actor object)
- **Outputs/Return:** None
- **Side effects:** Waits for ServerCommandStatus==cs_ready, calls ProcessPlayerCommand
- **Calls:** ProcessPlayerCommand, UpdateClientControls, M_LINKSTATE macro
- **Notes:** Waits with timeout (NETWORKTIMEOUT/MODEMTIMEOUT); handles FL_PUSHED flag

### SendFullServerPacket
- **Signature:** `void SendFullServerPacket(void)`
- **Purpose:** Bundle all clients' latest commands in COM_SERVER packet, broadcast
- **Inputs:** Global state (serverupdatetime, numplayers, ClientCommand)
- **Outputs/Return:** None
- **Side effects:** Constructs server packet, broadcasts to all, watches for quit/endgame
- **Calls:** BroadcastServerPacket, ResetClientCommands, GetPacketSize, memcpy
- **Notes:** Server-only; updates PlayerStatus on COM_QUIT/COM_ENDGAME

### AddClientPacket
- **Signature:** `void AddClientPacket(void * pkt, int src)`
- **Purpose:** Store received client packet in player command queue
- **Inputs:** pkt (MoveType or control packet), src (player index)
- **Outputs/Return:** None
- **Side effects:** Copies packet to PlayerCommand[src][time]
- **Calls:** memcpy, GetPacketSize, ProcessSoundAndDeltaPacket
- **Notes:** Special handling for COM_SOUNDANDDELTA to extract audio

### UpdatePlayerObj
- **Signature:** `void UpdatePlayerObj(int player)`
- **Purpose:** Unpack MoveType into PLAYERSTATE momentum, angle, buttons
- **Inputs:** player (index)
- **Outputs/Return:** None
- **Side effects:** Updates PLAYERSTATE[player] dmomx, dmomy, angle, button states
- **Calls:** MaxSpeedForCharacter, memset
- **Notes:** Called when COM_DELTA ready; extracts button bits into buttonstate array

### RecordDemo / LoadDemo / SetupDemo
- **Signature:** `void RecordDemo(void)`, `void LoadDemo(int demonumber)`, `void SetupDemo(void)`
- **Purpose:** Record gameplay or load/setup demo playback
- **Inputs:** demonumber (0-9 enum for demo file)
- **Outputs/Return:** None
- **Side effects:** Allocates demobuffer, sets demorecord/demoplayback flags, saves gamestate header
- **Calls:** SafeMalloc, LoadFile, SaveFile, InitializeWeapons, ResetPlayerstate, InitCharacter
- **Notes:** LoadDemo checks violence compatibility; SetupDemo copies header back to gamestate

**Additional minor functions** (ProcessPlayerCommand, ResendLocalPackets, CheckForPacket, RequestPacket, SetGameDescription, etc.) handle specific packet types, loss recovery, and game setup handshake.

## Control Flow Notes

**Init:**
1. `InitializeGameCommands()` → allocate buffers, set roles
2. `StartupClientControls()` → sync timing
3. `SetupGameMaster()`/`SetupGamePlayer()` → exchange config
4. `ServerLoop()` (server) → wait clients ready

**Per-frame loop:**
- `UpdateClientControls()` → poll input, create packets, advance controlupdatetime
- `ProcessServer()` (server) → wait all clients ready, send aggregated packet
- `ControlPlayerObj()` per player → wait for command ready, apply via `ProcessPlayerCommand()`

**Packet flow:**
Client sends → Server receives/stores → Server broadcasts → All clients apply commands

## External Dependencies
- **I/O:** `<dos.h>`, `<fcntl.h>`, `<io.h>`; ReadPacket, WritePacket (rt_com.h)
- **Memory:** SafeMalloc, SafeLevelMalloc, SafeFree (z_zone.h)
- **Timer:** CalcTics, ISR_SetTime (isr.h); ticcount, oldtime
- **Input:** PollControls, INL_GetMouseDelta, Keyboard[] (rt_util.h)
- **Game state:** gamestate, PLAYERSTATE[], PLAYER[], consoleplayer, characters[]
- **Sound:** SD_Play, SD_GetSoundData (rt_sound.h)
- **Battle:** BATTLE_GetOptions, AssignTeams, BATTLE_Team[] (rt_battl.h)
- **Debug:** Error, SoftError (rt_debug.h); AbortCheck
- **Network:** rottcom, ROTTpacket[], MAXCOMBUFFERSIZE (rottnet.h)
