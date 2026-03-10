# audiolib/source/dma.h

## File Purpose
Public header for DMA (Direct Memory Access) operations, providing an abstraction layer for setting up and monitoring memory-to-device transfers. Part of the audio library, likely used for real-time audio data streaming to sound hardware.

## Core Responsibilities
- Define error codes and transfer mode constants for DMA operations
- Declare channel verification and setup functions
- Declare transfer control and monitoring functions
- Provide error-to-string conversion for diagnostics

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `DMA_ERRORS` | enum | Error return codes (DMA_Error, DMA_Ok, DMA_ChannelOutOfRange, DMA_InvalidChannel) |
| `DMA_Modes` | enum | Transfer modes (SingleShot/AutoInit, Read/Write) |

## Global / File-Static State
None.

## Key Functions / Methods

### DMA_VerifyChannel
- Signature: `int DMA_VerifyChannel(int channel)`
- Purpose: Validate that a DMA channel number is within valid range
- Inputs: `channel` – DMA channel identifier
- Outputs/Return: DMA_ERRORS code (DMA_Ok if valid, error otherwise)
- Side effects: None
- Calls: Not inferable from header
- Notes: Likely performs bounds checking on channel ID

### DMA_SetupTransfer
- Signature: `int DMA_SetupTransfer(int channel, char *address, int length, int mode)`
- Purpose: Configure and initiate a DMA transfer operation
- Inputs: `channel` – DMA channel; `address` – memory buffer pointer; `length` – byte count; `mode` – DMA_Modes value
- Outputs/Return: DMA_ERRORS code
- Side effects: Programs DMA hardware registers; initiates memory transfer
- Calls: Likely calls DMA_VerifyChannel internally
- Notes: Address and length must be valid; mode determines transfer direction and repetition

### DMA_EndTransfer
- Signature: `int DMA_EndTransfer(int channel)`
- Purpose: Halt an active DMA transfer and clean up channel state
- Inputs: `channel` – DMA channel to stop
- Outputs/Return: DMA_ERRORS code
- Side effects: Stops DMA hardware; may assert interrupt disabling
- Calls: Not inferable

### DMA_GetCurrentPos
- Signature: `char *DMA_GetCurrentPos(int channel)`
- Purpose: Query current position within the transfer buffer
- Inputs: `channel` – DMA channel
- Outputs/Return: Pointer to current buffer position
- Side effects: Reads hardware registers
- Notes: Useful for determining audio playback position

### DMA_GetTransferCount
- Signature: `int DMA_GetTransferCount(int channel)`
- Purpose: Retrieve remaining bytes to transfer
- Inputs: `channel` – DMA channel
- Outputs/Return: Byte count remaining
- Side effects: Reads hardware registers
- Notes: Commonly polled to detect transfer completion

## Control Flow Notes
Header only; implements the public interface. Likely called during audio subsystem initialization (DMA_SetupTransfer), audio frame updates (DMA_GetCurrentPos/DMA_GetTransferCount polls), and shutdown (DMA_EndTransfer). Error codes suggest defensive channel validation on every operation.

## External Dependencies
None—self-contained header with no includes or external symbol references.
