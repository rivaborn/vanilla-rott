# rott/_rt_ser.h

## File Purpose
Private header defining UART 8250 serial port register offsets, control flags, and a circular queue structure for buffering serial data. Declares low-level serial I/O functions and CPU interrupt control macros used for modem-based network communication in the game.

## Core Responsibilities
- Define 8250 UART register addresses and bit flags for interrupt, FIFO, line control, modem status
- Define circular queue structure (`que_t`) for buffering transmitted/received serial data
- Declare serial port lifecycle functions (init, shutdown, enable)
- Declare byte-level read/write functions and interrupt service routine
- Provide low-level I/O macros for port access and CPU interrupt control

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `que_t` | struct | Circular queue with head/tail indices, size, and 2048-byte data buffer for serial data buffering |

## Global / File-Static State
None (header-only declarations).

## Key Functions / Methods

### InitPort
- Signature: `void InitPort(void)`
- Purpose: Initialize the serial port and UART controller
- Inputs: None
- Outputs/Return: None
- Side effects: Configures UART registers, likely sets up interrupt handlers
- Calls: Not visible in this file (defined elsewhere)
- Notes: Called during game startup for modem connection

### ShutdownPort
- Signature: `void ShutdownPort(void)`
- Purpose: Gracefully shut down the serial port
- Inputs: None
- Outputs/Return: None
- Side effects: Disables UART interrupts, restores interrupt vectors
- Calls: Not visible in this file
- Notes: Cleanup before exit

### isr_8250
- Signature: `void interrupt isr_8250(void)`
- Purpose: Interrupt service routine for 8250 UART events (RX data, TX ready, line/modem status)
- Inputs: None (triggered by hardware interrupt)
- Outputs/Return: None
- Side effects: Reads/writes UART registers, updates queue state, sends EOI to interrupt controller
- Calls: Not visible in this file
- Notes: Must be extremely fast; likely manipulates queue head/tail indices

### read_byte, write_byte, write_bytes, write_buffer
- Signature: `int read_byte(void)`, `void write_byte(unsigned char c)`, `void write_bytes(char *buf, int count)`, `void write_buffer(char *buffer, unsigned int count)`
- Purpose: Queue-based buffering for serial I/O (decouple application from UART timing)
- Inputs: Character(s) to transmit or buffer for output
- Outputs/Return: Byte read from queue (or -1 for empty)
- Side effects: Modify queue indices, may trigger UART transmission
- Calls: Not visible in this file
- Notes: Likely protected by CLI/STI interrupt disable/enable

### jump_start, EnablePort
- Signature: `void jump_start(void)`, `void EnablePort(void)`
- Purpose: Kickstart serial transmission and enable port operation
- Inputs: None
- Outputs/Return: None
- Side effects: Trigger UART transmission, enable FIFO/interrupts
- Calls: Not visible in this file
- Notes: Called after writing data to queue or after port initialization

## Control Flow Notes
This file is a foundational layer for modem-based multiplayer. The ISR (`isr_8250`) runs at interrupt level and buffers incoming/outgoing serial data. The application calls `write_byte(s)` at frame/tick level, and polling or event-driven code calls `read_byte()` to receive network packets. `InitPort()` runs during game startup, `ShutdownPort()` during shutdown.

## External Dependencies
- **Include**: `rottnet.h` (defines `MAXPACKETSIZE`, `MAXCOMBUFFERSIZE`, `rottcom_t` structure)
- **Defined elsewhere**: 
  - `inp()`, `outp()` — low-level port I/O (DOS/x86 intrinsics)
  - `_disable()`, `_enable()` — CPU interrupt control (Watcom/DOS intrinsics)
  - Interrupt vector setup and EOI handling (assumed in ISR implementation)
