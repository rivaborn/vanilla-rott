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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `setupdata_t` | struct | Setup packet payload: client flag, player number, command type, extra data, player count |
| `clientstate[]` | int array | Per-player state tracking (NoResponse / Echoed / Done) |
| `StringUsed[]` | boolean array | Tracks which humorous messages have been displayed |
| `NetStrings[]` | char* array | 25 humorous messages shown during player wait timeouts |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `numnetnodes` | int | global | Expected number of network nodes for this game |
| `socketid` | unsigned | global | IPX socket ID (default 0x882a = official ROTT socket) |
| `server` | boolean | global | True if running as server mode |
| `standalone` | boolean | global | True if server runs without participating as a player |
| `master` | boolean | global | Master node flag for game priority |
| `nodesetup` | setupdata_t | global | Local node's setup configuration packet |
| `playernumber` | int | global | Counter for next player number to assign |
| `clientstate[]` | int array | global | Per-player connection state (indices 0 to MAXNETNODES) |
| `messagesused` | int | static | Count of unique messages already displayed |
| `StringUsed[]` | boolean array | static | Bitmap of which messages have been shown |
| `NetStrings[]` | char* array | static | 25 hardcoded humorous waiting messages |

## Key Functions / Methods

### main
- **Signature:** `void main(void)`
- **Purpose:** Entry point; parses command-line arguments, initializes network, orchestrates node discovery, and launches the game
- **Inputs:** Command-line arguments via `_argc` / `_argv` (flags: `-server`, `-standalone`, `-master`, `-nodes N`, `-socket ADDR`, `-pause`, `-remoteridicule`)
- **Outputs/Return:** None (exits with `exit(0)`)
- **Side effects:** Initializes global state (`numnetnodes`, `server`, `standalone`, `master`, `socketid`, `rottcom`); calls `InitNetwork()`, `LookForNodes()`, `LaunchROTT()`, `Shutdown()`
- **Calls:** `CheckParm()`, `atoi()`, `InitNetwork()`, `LookForNodes()`, `LaunchROTT()`, `Shutdown()`
- **Notes:** Defaults to 2 nodes, non-server, non-standalone mode

### LookForNodes
- **Signature:** `void LookForNodes(void)`
- **Purpose:** Main node discovery and synchronization loop; handles server/client handshake to find all players and assign player numbers
- **Inputs:** Global state (`server`, `standalone`, `numnetnodes`)
- **Outputs/Return:** None; updates global `rottcom` and `clientstate[]` on completion
- **Side effects:** Sends/receives network packets, prints discovery progress and status messages, modifies `playernumber`, `clientstate[]`, `nodeadr[]`; resets on Escape key
- **Calls:** `ResetNodeAddresses()`, `SetupRandomMessages()`, `GetPacket()`, `SendPacket()`, `gettime()`, `bioskey()`, `Error()`, `PrintRandomMessage()`, `GetRandomMessage()`
- **Notes:** 
  - **Server mode:** broadcasts `cmd_FindClient` to all, receives `cmd_HereIAm` responses, assigns numbers via `cmd_YouAre`, waits for `cmd_IAm` confirmation
  - **Client mode:** listens for `cmd_FindClient`, responds with `cmd_HereIAm`, stores server address, receives `cmd_YouAre` with player number, confirms with `cmd_IAm`
  - Timeout: retries every `MAXWAIT` (10) seconds with random humorous message
  - Abort condition: Escape key or all expected players connected and confirmed
  - Detects conflicts: multiple servers, unknown game packets, incorrect player numbers

### ResetNodeAddresses
- **Signature:** `void ResetNodeAddresses(void)`
- **Purpose:** Reset node address table and assign initial player number (used when master node joins)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `clientstate[]`, `nodeadr[]` arrays; resets `playernumber` to 1; copies local address to `nodeadr[1]` if not standalone
- **Calls:** `memset()`, `memcpy()`
- **Notes:** Called once at startup and again if a master client joins later

### Shutdown
- **Signature:** `void Shutdown(void)`
- **Purpose:** Cleanup and teardown of network and communication modules
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `ShutdownROTTCOM()` and `ShutdownNetwork()`
- **Calls:** `ShutdownROTTCOM()`, `ShutdownNetwork()`
- **Notes:** Invoked just before program exit

### NetISR
- **Signature:** `void NetISR(void)`
- **Purpose:** Interrupt Service Routine (ISR) for asynchronous network events (likely hooked by IPX driver)
- **Inputs:** None (checks global `rottcom.command`)
- **Outputs/Return:** None
- **Side effects:** Increments `ipxlocaltime` if sending; calls `SendPacket()` or `GetPacket()`
- **Calls:** `SendPacket()`, `GetPacket()`
- **Notes:** Triggered by IPX driver; disambiguates send vs. receive commands

### SetupRandomMessages
- **Signature:** `void SetupRandomMessages(void)`
- **Purpose:** Initialize random message display tracking and randomizer
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resets `messagesused` to 0, clears `StringUsed[]` array, calls `randomize()`
- **Calls:** `memset()`, `randomize()`
- **Notes:** Called at start of `LookForNodes()` and when all 25 messages have been cycled

### GetRandomMessage
- **Signature:** `int GetRandomMessage(void)`
- **Purpose:** Return a random unused message index, cycling through all 25 before repeating
- **Inputs:** None
- **Outputs/Return:** Integer index [0, MAXNETMESSAGES-1] to `NetStrings[]`
- **Side effects:** Updates `messagesused` and `StringUsed[]` array; calls `SetupRandomMessages()` if all used
- **Calls:** `random()`, `SetupRandomMessages()`
- **Notes:** Ensures each message is shown exactly once per cycle; resets cycle when exhausted

### PrintRandomMessage
- **Signature:** `void PrintRandomMessage(int message, int player)`
- **Purpose:** Format and display a humorous waiting message with player number substitution
- **Inputs:** `message` (index into `NetStrings[]`), `player` (player number for %d substitution)
- **Outputs/Return:** None (prints to stdout via `printf`)
- **Calls:** `printf()`
- **Notes:** Simple wrapper; no range checking on `message` index

## Control Flow Notes

**Initialization Phase (main):**
1. Parse command line for mode flags and node count
2. Call `InitNetwork()` to set up IPX driver
3. Enter `LookForNodes()` for discovery and synchronization
4. Mark setup complete (`ipxlocaltime = 0`)
5. Call `LaunchROTT()` to start the game
6. On game exit, call `Shutdown()` and exit

**Node Discovery (LookForNodes):**
- **If server:** Broadcast requests, collect responses, assign numbers, wait for confirmations
- **If client:** Await broadcast, identify self, receive assignment
- Both modes validate packet consistency and detect conflicts
- 10-second timeout loop displays humorous messages to user

**Networking:**
The `NetISR()` ISR is hooked by the IPX driver to handle async send/receive. The main thread polls via `GetPacket()` in `LookForNodes()`.

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
