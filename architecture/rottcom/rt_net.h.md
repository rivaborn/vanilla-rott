# rottcom/rt_net.h

## File Purpose
Defines network packet types and structures for multiplayer game communication. Provides type definitions for all network messages (movement, game state, chat, synchronization) and utility functions to calculate packet sizes for serialization/deserialization.

## Core Responsibilities
- Define network command type constants (COM_DELTA, COM_TEXT, COM_SYNC, etc.)
- Provide packet structure definitions for each message type
- Calculate packet sizes dynamically based on packet type
- Handle server multi-packet aggregation
- Define audio transmission and synchronization check structures

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| DemoType | struct | Single frame delta: time, momentum, angle, buttons |
| MoveType | struct | Movement packet with embedded sound data |
| NullMoveType | struct | Minimal null movement (type + time only) |
| COM_ServerHeaderType | struct | Server packet wrapper for multiple sub-packets |
| COM_RequestType | struct | Request for missing packets |
| COM_TextType | struct | Chat message with recipient |
| COM_RemoteRidiculeType | struct | Remote ridicule sound effect and player info |
| COM_FixupType | struct | Correction/fixup packet |
| COM_SyncType | struct | Time synchronization packet |
| COM_CheckSyncType | struct | Detailed sync check with position/state |
| COM_GameMasterType | struct | Master game configuration (level, violence, players, settings) |
| COM_GamePlayerType | struct | Player description (character, color, codename) |
| COM_GameAckType | struct | Game acknowledgment |
| COM_PauseType, COM_UnPauseType | struct | Pause/unpause commands |
| COM_RespawnType | struct | Player respawn command |
| CommandType | struct | Array of 256 command pointers |
| CommandStatusType | struct | Array of 256 command state bytes |

## Global / File-Static State
None.

## Key Functions / Methods

### GetPacketSize
- **Signature:** `int GetPacketSize(void * pkt)`
- **Purpose:** Determine the serialized size of a network packet based on its type field
- **Inputs:** `pkt` — generic packet pointer (cast to MoveType to read type field)
- **Outputs/Return:** Packet size in bytes
- **Side effects:** Calls Error() if packet type is unhandled (fatal)
- **Calls:** `sizeof()` on various COM_*Type structures
- **Notes:** Switch on packet type; COM_SERVER subtracts 1 byte; COM_SOUNDANDDELTA combines MoveType + COM_SoundType sizes. Missing case returns error.

### GetServerPacketSize
- **Signature:** `int GetServerPacketSize(void * pkt)`
- **Purpose:** Calculate total size of a server packet that may contain multiple nested sub-packets
- **Inputs:** `pkt` — packet pointer (cast to COM_ServerHeaderType)
- **Outputs/Return:** Total size from packet start to end of all sub-packets
- **Side effects:** Iterates through numpackets field
- **Calls:** `GetPacketSize()` for each sub-packet
- **Notes:** If type is COM_SERVER, walks the data array and accumulates sizes of each sub-packet; otherwise delegates to GetPacketSize(). Pointer arithmetic: `ptr - pkt` returns byte offset.

## Control Flow Notes
This header defines the data structures used in the network loop (send/receive in multiplayer). Packets flow bidirectionally: local input → movement/command packets sent to remote nodes, remote packets received and deserialized. GetPacketSize() is called during serialization (write) and deserialization (read) to handle variable-length messages. Server packets aggregate multiple client packets for efficiency.

## External Dependencies
- **Includes:** `rottnet.h` (defines MAXPLAYERS, MAXNETNODES, MAXCODENAMELENGTH, rottcom_t structure)
- **Types used:** `gametype`, `specials`, `battle_type`, `word`, `byte`, `boolean` (defined elsewhere)
- **Constants:** `DUMMYPACKETSIZE`, `MAXCODENAMELENGTH`, `COM_SOUND_BUFFERSIZE` (256), `COM_MAXTEXTSTRINGLENGTH` (33)
- **Notes:** Conditional COM_SYNCCHECK define gated on SYNCCHECK macro; allows optional sync validation
