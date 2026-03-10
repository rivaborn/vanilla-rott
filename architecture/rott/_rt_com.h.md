# rott/_rt_com.h

## File Purpose
Private header file defining network synchronization packet structures and constants for multiplayer/networked gameplay. Establishes the protocol for synchronizing game state between networked instances using a multi-phase handshake.

## Core Responsibilities
- Define synchronization packet structure and format
- Establish phase constants for multi-stage synchronization protocol
- Define timing and sizing parameters for sync operations
- Provide metadata wrapper around sync packets

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `syncpackettype` | struct | Network packet containing synchronization data with type identifier, phase info, timing, and payload |
| `synctype` | struct | Wrapper around a sync packet with send/delta timing metadata |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SYNCPACKETSIZE` | #define (int) | global | Maximum payload size for sync packet data (32 bytes) |
| `SYNC_PHASE0` through `SYNC_PHASE5`, `SYNC_MEMO` | #define (int) | global | Phase identifiers for synchronization handshake states |
| `NUMSYNCPHASES` | #define (int) | global | Total number of active sync phases (5) |
| `SYNCTIME` | #define (int) | global | Timing constant for sync operations (15 units) |

## Key Functions / Methods
None (header-only; structure and constant definitions only).

## Control Flow Notes
This file establishes the protocol for networked game synchronization. The multi-phase approach (`SYNC_PHASE0` through `SYNC_PHASE5`) suggests a handshake-based synchronization model, likely used during:
- Multiplayer game initialization
- Player join/sync operations
- Periodic state reconciliation

The `SYNC_MEMO` phase (value 99) appears to be a special case, possibly for memoization or snapshot operations.

## External Dependencies
- **C standard types**: `byte`, `int` (language primitives; assume standard integer definitions elsewhere in codebase)
- No explicit includes or external symbol dependencies visible in this file
