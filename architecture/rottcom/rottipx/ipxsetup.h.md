# rottcom/rottipx/ipxsetup.h

## File Purpose
Header file for IPX network setup and initialization in the ROTT game engine. Declares global network state variables and a shutdown routine for the IPX (Internetwork Packet Exchange) networking layer.

## Core Responsibilities
- Export IPX socket identifier for network communication
- Export server/client mode flag
- Export network node count state
- Declare shutdown procedure for network cleanup

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `socketid` | unsigned | global | IPX socket identifier for network communication |
| `server` | boolean | global | Flag indicating whether this instance is a network server |
| `numnetnodes` | int | global | Count of active network nodes in the session |

## Key Functions / Methods

### Shutdown
- Signature: `void Shutdown(void)`
- Purpose: Clean shutdown of IPX networking layer
- Inputs: None
- Outputs/Return: None
- Side effects: Likely closes socket, cleans up network resources (implementation not visible here)
- Calls: Not inferable from this file
- Notes: Presumably called during game shutdown or when exiting network play

## Control Flow Notes
This header supports network initialization/shutdown. The `socketid`, `server`, and `numnetnodes` variables are likely initialized early in the game setup phase and used throughout multiplayer sessions. `Shutdown()` is called at game exit.

## External Dependencies
- Standard C types (`unsigned`, `int`, `boolean`, `void`)
- Definitions of `boolean` and related types come from elsewhere in the codebase
