# rottcom/rottser/port.h

## File Purpose
Defines the serial port communication interface for DOS-era hardware, including circular queue data structures for buffered I/O and interrupt service routine declarations for 8250/16550 UART chips. Enables non-blocking serial communication via hardware interrupts.

## Core Responsibilities
- Define circular queue structure (`que_t`) for input/output buffering with power-of-2 sizing
- Declare global configuration variables (IRQ, UART type, COM port, baud rate)
- Declare interrupt service routines (ISRs) for 8250 and 16550 UART chip types
- Declare initialization/shutdown functions for serial port hardware setup
- Provide byte-level and buffer-level read/write functions for queued serial I/O

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| que_t | struct | Circular queue with head/tail indices, size field, and fixed 2048-byte buffer for serial data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| inque | que_t | extern global | Input queue for received serial data |
| outque | que_t | extern global | Output queue for data to transmit |
| irq | int | extern global | Interrupt request number assigned to serial port |
| uart | int | extern global | UART chip type identifier (8250 or 16550) |
| comport | int | extern global | COM port number (1–4) |
| baudrate | long | extern global | Baud rate configuration |

## Key Functions / Methods

### InitPort
- Signature: `void InitPort(void)`
- Purpose: Initialize serial port hardware and install interrupt handlers
- Inputs: None
- Outputs/Return: None
- Side effects: Configures UART, installs ISR, initializes queues
- Calls: Not inferable from this file
- Notes: Called during startup to enable serial communication

### ShutdownPort
- Signature: `void ShutdownPort(void)`
- Purpose: Shut down serial port and restore original interrupt handlers
- Inputs: None
- Outputs/Return: None
- Side effects: Uninstalls ISR, restores system state
- Calls: Not inferable from this file
- Notes: Called during cleanup/exit

### jump_start
- Signature: `void jump_start(void)`
- Purpose: Start/resume serial communication after initialization
- Inputs: None
- Outputs/Return: None
- Side effects: May trigger transmission or enable interrupts
- Calls: Not inferable from this file

### GetUart
- Signature: `void GetUart(void)`
- Purpose: Detect and identify UART chip type on system
- Inputs: None
- Outputs/Return: None (sets global `uart`)
- Side effects: Probes hardware, modifies `uart` global
- Calls: Not inferable from this file

### read_byte
- Signature: `int read_byte(void)`
- Purpose: Read one byte from input queue
- Inputs: None
- Outputs/Return: Byte value or error code
- Side effects: Advances `inque.tail`
- Calls: Not inferable from this file
- Notes: Non-blocking; returns immediately

### write_byte, write_bytes, write_buffer
- Signatures: `void write_byte(unsigned char c)`, `void write_bytes(char *buf, int count)`, `void write_buffer(char *buffer, unsigned int count)`
- Purpose: Queue bytes for serial transmission
- Side effects: Advance `outque.head`, may trigger ISR transmission
- Notes: Wrappers with varying buffer/count parameter styles

### isr_8250, isr_16550
- Signatures: `void interrupt isr_8250(void)`, `void interrupt isr_16550(void)`
- Purpose: Hardware interrupt handlers for UART data/status events
- Inputs: None (hardware-triggered)
- Outputs/Return: None (interrupt handlers)
- Side effects: Read/write UART I/O ports, modify `inque`/`outque`
- Notes: DOS-era `interrupt` keyword; 16550 has FIFO buffering vs. 8250 single-byte

## Control Flow Notes
Initialization chain: `GetUart()` → `InitPort()` → `jump_start()` → serial I/O active. During operation, hardware interrupts invoke ISRs to transfer data between UART and queues. Application calls `read_byte()`/`write_byte*()` for non-blocking queue access. Shutdown calls `ShutdownPort()` to restore system state.

## External Dependencies
- `<conio.h>`, `<dos.h>` — DOS/BIOS console and system interrupt support (legacy, requires DOS or DPMI mode)
- Hardware: 8250 or 16550 UART chip accessible via I/O port addresses
- Macro `QueSpot(index)` implements power-of-2 circular queue modulo using bitwise AND
