# rott/rt_net.h

## File Purpose
Network protocol and command management header for ROTT's multiplayer system. Defines packet structures, command types, and function interfaces for synchronizing game state, player input, and demo recording across networked game instances (both modem and LAN).

## Core Responsibilities
- Define network command protocol (25+ command types: delta, sync, pause, respawn, text messages, etc.)
- Declare packet structures for all network message types (player descriptions, game config, synchronization)
- Manage demo recording/playback system integration with network
- Coordinate client-side input capture and server-side distribution
- Handle player description and game configuration exchange at startup
- Implement remote ridicule (voice/text) transmission between players
- Provide synchronization checks and timeout mechanisms

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| DemoType | struct | Single frame of demo data: position, angle, buttons, timestamp |
| DemoHeaderType | struct | Demo file header containing game state snapshot |
| MoveType | struct | Player input + movement for one frame (includes variable sound data) |
| NullMoveType | struct | Minimal null movement record (type + timestamp only) |
| COM_SyncType | struct | Synchronization packet (type, server time) |
| COM_CheckSyncType | struct | Sync verification packet (position, angle, random seed, player state) |
| COM_ServerHeaderType | struct | Server packet wrapper (type, time, packet count, data) |
| COM_TextType | struct | Text message (max 33 chars, recipient, timestamp) |
| COM_RemoteRidiculeType | struct | Remote ridicule command (player, sound ID, recipient) |
| COM_PlayerDescriptionType | struct | Player setup (character, color, codename) |
| COM_GamePlayerType | struct | Game player initialization packet |
| COM_GameMasterType | struct | Game master config (level, violence, players array, options) |
| COM_GameAckType | struct | Game acknowledgment from client |
| COM_PauseType / COM_UnPauseType | struct | Pause/unpause state change |
| COM_RespawnType | struct | Respawn command |
| CommandType | struct | Array of 256 command pointers |
| CommandStatusType | struct | Array of 256 command state bytes |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| demorecord | boolean | extern | Flag: currently recording demo |
| demoplayback | boolean | extern | Flag: currently playing back demo |
| demoptr, lastdemoptr | byte* | extern | Demo buffer read/write pointers |
| demobuffer | byte* | extern | Demo recording buffer |
| demodone | boolean | extern | Flag: demo playback finished |
| IsServer | boolean | extern | Flag: this console is server |
| standalone | boolean | extern | Flag: single-player mode |
| playerdead | boolean | extern | Flag: local player is dead |
| modemgame, networkgame | boolean | extern | Game connection type flags |
| numplayers | int | extern | Active player count |
| server | int | extern | Server player index |
| GamePaused | boolean | extern | Flag: game is paused |
| battlegibs | boolean | extern | Flag: gibs enabled in battle |
| remoteridicule | boolean | extern | Flag: remote ridicule active |
| lastsynccheck | int | extern | Last sync check timestamp (SYNCCHECK=1 only) |
| PlayerSync | COM_CheckSyncType[] | extern | Per-player sync state array (SYNCCHECK=1 only) |

## Key Functions / Methods

### ControlPlayer
- Signature: `void ControlPlayer(void)`
- Purpose: Process input from local human player, apply to player object
- Calls: (not visible in header)
- Notes: Core per-frame input handling

### ControlRemote
- Signature: `void ControlRemote(objtype * ob)`
- Purpose: Apply networked input to remote player object
- Inputs: `ob` – remote player object
- Notes: Applies received command data to non-local player

### UpdateClientControls
- Signature: `void UpdateClientControls(void)`
- Purpose: Distribute local player input to server/network
- Side effects: Updates network buffers with player commands
- Notes: Called once per frame for client-side sync

### ProcessServer
- Signature: `void ProcessServer(void)`
- Purpose: Receive and distribute server commands to all local actors
- Side effects: Updates local game state from server packets
- Notes: Server-side frame dispatch

### ServerLoop
- Signature: `void ServerLoop(void)`
- Purpose: Main server-side frame loop iteration
- Side effects: Coordinates server command processing and distribution
- Notes: Called once per network frame

### SendPlayerDescription
- Signature: `void SendPlayerDescription(void)`
- Purpose: Transmit local player setup to other players
- Side effects: Sends COM_PlayerDescriptionType packet

### SendGameDescription
- Signature: `void SendGameDescription(void)`
- Purpose: Transmit game configuration from server to clients
- Side effects: Sends COM_GameMasterType packet

### SetGameDescription
- Signature: `void SetGameDescription(void * pkt)`
- Purpose: Parse and apply received game config packet
- Inputs: `pkt` – pointer to COM_GameMasterType data

### SendGameAck
- Signature: `void SendGameAck(void)`
- Purpose: Client acknowledges game setup complete
- Side effects: Sends COM_GameAckType packet

### RecordDemo
- Signature: `void RecordDemo(void)`
- Purpose: Record current frame to demo buffer
- Side effects: Writes to demobuffer, advances demoptr
- Notes: Frame data stored as DemoType + optional sound

### LoadDemo
- Signature: `void LoadDemo(int demonumber)`
- Purpose: Load demo from disk into memory
- Inputs: `demonumber` – demo slot (0-N)
- Side effects: Populates demobuffer, sets playback state

### SaveDemo
- Signature: `void SaveDemo(int demonumber)`
- Purpose: Write in-memory demo to disk
- Inputs: `demonumber` – demo slot to save to

### AddTextMessage
- Signature: `void AddTextMessage(char * message, int length, int towho)`
- Purpose: Queue text message for transmission
- Inputs: `message` – text to send, `length` – byte count, `towho` – recipient (255=all, 254=team)

### AddRemoteRidiculeCommand
- Signature: `void AddRemoteRidiculeCommand(int player, int towho, int num)`
- Purpose: Queue remote ridicule (sound effect) command
- Inputs: `player` – source, `towho` – target player(s), `num` – sound ID

### ProcessRemoteRidicule
- Signature: `void ProcessRemoteRidicule(void * pkt)`
- Purpose: Receive and play remote ridicule packet
- Inputs: `pkt` – COM_RemoteRidiculeType pointer

### SyncToServer
- Signature: `void SyncToServer(void)`
- Purpose: Send synchronization request to server; used for lag/latency management
- Side effects: Transmits COM_SyncType packet

### CheckForSyncCheck
- Signature: `void CheckForSyncCheck(void)` (SYNCCHECK=1 only)
- Purpose: Verify game state consistency across network
- Side effects: Compares local PlayerSync[] with received data; logs/handles desync

### GamePacketSize
- Signature: `int GamePacketSize(void)`
- Outputs/Return: Size of next outgoing network packet in bytes

### PlayerInGame
- Signature: `boolean PlayerInGame(int p)`
- Inputs: `p` – player index
- Outputs/Return: true if player is active in current game

### ConsoleIsServer
- Signature: `boolean ConsoleIsServer(void)`
- Outputs/Return: true if this console is running the server

### DemoExists
- Signature: `boolean DemoExists(int demonumber)`
- Inputs: `demonumber` – demo slot
- Outputs/Return: true if demo file exists on disk

**Notes on trivial helpers:** `AddEndGameCommand`, `AddPauseStateCommand`, `AddQuitCommand`, `AddExitCommand`, `AddGameEndCommand`, `AddRespawnCommand`, `ResetCurrentCommand`, `SetupGamePlayer`, `SetupGameMaster`, `SetNormalHorizon`, `SetupDemo`, `FreeDemo`, `InitializeGameCommands`, `ShutdownGameCommands`, `StartupClientControls`, `ShutdownClientControls` — all command/setup utilities; declarations only.

## Control Flow Notes
This file is integrated into the main game loop's **update phase**:
- **Client side:** `UpdateClientControls()` → capture local input → queue commands → `ProcessServer()` distributes server packets
- **Server side:** `ServerLoop()` → `ProcessServer()` receives all client input → distributes commands to `ControlRemote()` for other players
- **Demo recording:** `RecordDemo()` called once per frame if `demorecord=true`; captures MoveType snapshot
- **Synchronization:** `SyncToServer()` and optional `CheckForSyncCheck()` run periodically (conditional SYNCCHECK=1)
- **Pause/respawn/exit:** Special commands (`AddPauseStateCommand`, etc.) queued and distributed to ensure all clients see same state

## External Dependencies
- **develop.h** – feature flags (SYNCCHECK, etc.)
- **rottnet.h** – low-level network driver interface (rottcom_t, modem/network game constants)
- **rt_actor.h** – object/actor structures (objtype, classtype)
- **rt_battl.h** – battle system types (battle_type, specials, battle_options)
- **rt_playr.h** – player state (playertype, MAXCODENAMELENGTH)
- **rt_main.h** – main game loop integration (VBLCOUNTER timing)
- **Defined elsewhere:** External symbols `VBLCOUNTER` (VBL tick counter), `MAXPLAYERS` (from rottnet.h), game state globals
