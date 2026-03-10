# audiolib/source/dma.c

## File Purpose
Low-level DMA controller driver for ISA bus 8-bit and 16-bit transfers. Provides hardware abstraction for configuring Intel 8237 DMA controller channels via I/O port programming, with error reporting and transfer status queries. Targets DOS/early Windows ISA architecture.

## Core Responsibilities
- Channel validation and error reporting for DMA operations
- Configure DMA controller with address, length, and transfer mode (single-shot/auto-init, read/write)
- Enable/disable DMA channels via hardware mask ports
- Query live transfer position and remaining byte count during active DMA
- Abstract 8-bit (channels 0–3) and 16-bit (channels 5–7) DMA port mappings

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `DMA_PORT` | struct | Maps a DMA channel to its hardware control port addresses (mask, mode, clear, page, address, length) and data width (BYTE/WORD). |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DMA_PortInfo` | `static const DMA_PORT[8]` | static | Hardware port address lookup table for all 8 DMA channels; includes validity flags and BYTE/WORD width. |
| `DMA_ErrorCode` | `int` | global | Current error state; updated by operations and queried via `DMA_ErrorString()`. |

## Key Functions / Methods

### DMA_ErrorString
- **Signature:** `char *DMA_ErrorString(int ErrorNumber)`
- **Purpose:** Convert error codes to human-readable error messages.
- **Inputs:** Error code (e.g., `DMA_Error`, `DMA_Ok`, `DMA_ChannelOutOfRange`, `DMA_InvalidChannel`).
- **Outputs/Return:** Pointer to static error string.
- **Side effects:** If `ErrorNumber == DMA_Error`, recursively fetches `DMA_ErrorCode` and looks up its string.
- **Calls:** Itself (recursive for `DMA_Error` case).
- **Notes:** Recursive call avoids extra parameter passing; "Unknown DMA error code" fallback prevents crashes on invalid codes.

### DMA_VerifyChannel
- **Signature:** `int DMA_VerifyChannel(int channel)`
- **Purpose:** Validate that a DMA channel is in range and marked as valid (not disabled).
- **Inputs:** Channel number (0–7).
- **Outputs/Return:** `DMA_Ok` on success; `DMA_Error` on failure; sets `DMA_ErrorCode`.
- **Side effects:** Updates global `DMA_ErrorCode`.
- **Calls:** `DMA_SetErrorCode()` macro.
- **Notes:** Channels 2 and 4 are marked `INVALID` in hardware table (typical for ISA DMA).

### DMA_SetupTransfer
- **Signature:** `int DMA_SetupTransfer(int channel, char *address, int length, int mode)`
- **Purpose:** Configure DMA controller for a data transfer on the specified channel.
- **Inputs:** Channel (0–7), memory address (16-bit or 20-bit), byte length, mode (read/write/auto-init).
- **Outputs/Return:** `DMA_Ok` or `DMA_Error` (from channel verification).
- **Side effects:** Writes to DMA controller I/O ports (mask, mode, address, page, length); programs hardware state.
- **Calls:** `DMA_VerifyChannel()`, `outp()` (hardware I/O).
- **Notes:** Derives 20-bit address from 16-bit pointer using page register. For WORD channels, shifts address right by 1 and adjusts length. DMA length register is pre-decremented (N−1 format).

### DMA_EndTransfer
- **Signature:** `int DMA_EndTransfer(int channel)`
- **Purpose:** Disable a DMA channel and reset its hardware state.
- **Inputs:** Channel number.
- **Outputs/Return:** `DMA_Ok` or `DMA_Error`.
- **Side effects:** Masks (disables) the DMA channel via hardware mask port; clears flip-flop.
- **Calls:** `DMA_VerifyChannel()`, `outp()`.
- **Notes:** Must be called after transfer completes or to abort; clearing flip-flop ensures next I/O is to low byte.

### DMA_GetCurrentPos
- **Signature:** `char *DMA_GetCurrentPos(int channel)`
- **Purpose:** Read the current memory address pointer of an active DMA transfer.
- **Inputs:** Channel number.
- **Outputs/Return:** Memory address (void pointer cast to `char*`); `NULL` on error.
- **Side effects:** Reads from DMA address and page port registers (hardware state, not modified).
- **Calls:** `DMA_VerifyChannel()`, `inp()` (hardware I/O).
- **Notes:** For WORD channels, reconstructs address by shifting low byte left 1 bit and high byte left 9 bits; page register provides bits 16–23. If channel invalid, returns `NULL`.

### DMA_GetTransferCount
- **Signature:** `int DMA_GetTransferCount(int channel)`
- **Purpose:** Query remaining byte count in an active DMA transfer.
- **Inputs:** Channel number.
- **Outputs/Return:** Remaining bytes; sets `DMA_ErrorCode`.
- **Side effects:** Reads DMA length register (16-bit, little-endian); clears flip-flop before read.
- **Calls:** `DMA_VerifyChannel()` (inlined logic), `outp()`, `inp()`.
- **Notes:** For WORD channels, left-shifts result by 1 to convert word count to byte count. Does not delegate verification to `DMA_VerifyChannel()` function—duplicates logic inline.

## Control Flow Notes
This module is **initialization/configuration code**. Typical usage:
1. `DMA_SetupTransfer()` once at start of audio/transfer operation.
2. `DMA_GetCurrentPos()` / `DMA_GetTransferCount()` polled during transfer to monitor progress.
3. `DMA_EndTransfer()` once transfer completes.

No frame loop integration; operates independently of render/update cycles.

## External Dependencies
- **`<dos.h>`** – `outp()`, `inp()` for 8086 ISA I/O port read/write.
- **`<conio.h>`** – Console/hardware I/O (may overlap with dos.h).
- **`<stdlib.h>`** – Standard C library.
- **`dma.h`** – Local header defining error and mode enums and public function signatures.

---

**Notes:**
- Author: James R. Dose (1994).
- Hard-coded 8237 DMA controller port addresses assume ISA bus (not PCI/modern).
- Hardware-specific: `0xA`, `0xB`, `0xC` for 8-bit; `0xD4`, `0xD6`, `0xD8` for 16-bit.
- Uses bitwise operations and shifts heavily; address reconstruction is non-obvious.
