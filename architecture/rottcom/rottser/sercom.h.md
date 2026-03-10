# rottcom/rottser/sercom.h

## File Purpose
Header declaring the serial communication interface for network gameplay in Rise of the Triad. Provides functions for packet transmission/reception, connection management, interrupt handling, and communication diagnostics.

## Core Responsibilities
- Declare packet-based network I/O functions (send/receive)
- Track serial line error statistics (break, framing, parity, overrun)
- Count interrupt events (Tx/Rx)
- Provide connection setup and interrupt service routine
- Offer timing measurement for network latency analysis
- Reset and report communication counters for debugging

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| numBreak | unsigned long | global | Break signal count |
| numFramingError | unsigned long | global | Serial framing error count |
| numParityError | unsigned long | global | Serial parity error count |
| numOverrunError | unsigned long | global | Receive buffer overrun count |
| numTxInterrupts | unsigned long | global | Transmission interrupt count |
| numRxInterrupts | unsigned long | global | Reception interrupt count |
| writeBufferOverruns | unsigned long | global | Transmit buffer overrun count |

## Key Functions / Methods

### Connect
- Signature: `int Connect(void);`
- Purpose: Establish network/serial connection
- Inputs: None
- Outputs/Return: int (likely 0 on success, negative on failure)
- Side effects: Initializes serial hardware or modem connection
- Calls: Not inferable from this file
- Notes: Called once at network session start

### WritePacket
- Signature: `void WritePacket(char *buffer, int len);`
- Purpose: Transmit packet to remote peer
- Inputs: buffer (packet data), len (length in bytes)
- Outputs/Return: None
- Side effects: Sends data over serial link; may trigger Tx interrupt
- Calls: Not inferable from this file

### ReadPacket
- Signature: `boolean ReadPacket(void);`
- Purpose: Poll for received packet
- Inputs: None
- Outputs/Return: boolean (true if packet available, false if empty)
- Side effects: Updates internal receive buffer; increments Rx counters
- Calls: Not inferable from this file
- Notes: Destination buffer passed separately (not visible in signature)

### NetISR
- Signature: `void NetISR(void);`
- Purpose: Interrupt service routine for serial communication events
- Inputs: None
- Outputs/Return: None
- Side effects: Processes incoming/outgoing data; increments interrupt counters; updates error flags
- Calls: Not inferable from this file
- Notes: Called asynchronously by hardware; must be fast

### reset_counters
- Signature: `void reset_counters(void);`
- Purpose: Zero all diagnostic counters
- Inputs: None
- Outputs/Return: None
- Side effects: Resets all global counter variables
- Calls: Not inferable from this file

### stats
- Signature: `void stats(void);`
- Purpose: Display current communication statistics to console
- Inputs: None
- Outputs/Return: None
- Side effects: I/O output (debug/profiling)
- Calls: Not inferable from this file

### StartTime / EndTime
- Signatures: `void StartTime(void);` / `void EndTime(void);`
- Purpose: Bracket network operations for latency measurement
- Inputs/Outputs: None
- Side effects: Records elapsed time between calls
- Calls: Not inferable from this file
- Notes: Paired calls; likely used to measure round-trip or packet handling latency

## Control Flow Notes
Typical game loop integration:
1. **Init**: `Connect()` establishes serial connection
2. **Frame loop**: `ReadPacket()` polls for incoming updates; `WritePacket()` sends local state
3. **Background**: `NetISR()` fires on hardware interrupt to handle serial events
4. **Debug**: `stats()` / `reset_counters()` called on demand for profiling
5. **Latency measurement**: `StartTime()`/`EndTime()` wrap critical network sections

The presence of low-level error counters (break, framing, parity, overrun) suggests direct serial hardware control—likely null-modem or modem-based multiplayer.

## External Dependencies
- Standard C (`void`, `int`, `char`, `unsigned long`, `boolean`)
- Serial hardware interrupt registration (implied by `NetISR`)
- Timing subsystem (implied by `StartTime`/`EndTime`)
- All implementations defined elsewhere
