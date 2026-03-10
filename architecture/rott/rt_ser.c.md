# rott/rt_ser.c

## File Purpose
Implements serial/modem communication for multiplayer networking in the ROTT game engine. Manages UART (8250/16550) hardware, interrupt-driven I/O queues, and frame-based packet encoding/decoding with escape sequence handling for reliable data transmission over serial links.

## Core Responsibilities
- UART hardware initialization, configuration (baud rate, interrupt vectors), and shutdown
- Interrupt service routine (ISR) for handling serial transmit/receive/modem status interrupts
- Ring buffers (queues) for buffering incoming and outgoing serial data
- Frame-based packet protocol with escape-character encoding to handle binary data
- Interactive "talk mode" for keyboard-to-serial communication
- Queue manipulation (read/write single bytes and multi-byte buffers)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `que_t` | struct | Ring buffer for serial I/O; head/tail pointers, size, data array |
| `serialdata_t` | struct (from rottser.h) | Configuration container with IRQ, UART base address, baud rate |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `localbuffer` | char[] | static | Temporary buffer for encoding packets with escape sequences |
| `serialpacket` | char[] | static | Holds current received packet data |
| `serialpacketlength` | int | static | Length of current packet being reassembled |
| `inque` | que_t | static | Input queue for received serial data |
| `outque` | que_t | static | Output queue for data to transmit |
| `uart` | int | static | Base I/O address of serial UART port |
| `irq` | int | static | IRQ number (3 or 4, typically) |
| `baudrate` | int | static | Baud rate setting |
| `uart_type` | enum | static | Detected UART type (UART_8250 or UART_16550) |
| `modem_status` | int | static | Last read modem status register value |
| `line_status` | int | static | Last read line status register value |
| `oldirqvect` | function pointer | static | Saved original interrupt vector (for restoration on shutdown) |
| `irqintnum` | int | static | Calculated interrupt number (irq + 8) |
| `inescape` | int | static | Frame parser state: true if last byte was escape char |
| `newpacket` | int | static | Frame parser state: signals start of new packet reassembly |

## Key Functions / Methods

### SetupModemGame
- **Signature:** `void SetupModemGame(void)`
- **Purpose:** Initialize modem for multiplayer game session; extract serial parameters from game config.
- **Inputs:** None (reads from global `rottcom->data[0]`)
- **Outputs/Return:** None (writes to global state: `irq`, `uart`, `baudrate`)
- **Side effects:** Calls `InitPort()` to configure hardware
- **Calls:** `InitPort()`
- **Notes:** Retrieves config from a presumed global `rottcom` structure; parameters are passed through static globals rather than return values

### ShutdownModemGame
- **Signature:** `void ShutdownModemGame(void)`
- **Purpose:** Clean up serial port on game exit.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `ShutdownPort()` to disable UART and restore interrupt vector
- **Calls:** `ShutdownPort()`
- **Notes:** Counterpart to `SetupModemGame()`

### InitPort
- **Signature:** `void InitPort(void)`
- **Purpose:** Initialize queues, detect UART type, and enable interrupts.
- **Inputs:** None (reads global `uart` and `baudrate`)
- **Outputs/Return:** None (initializes global `inque`, `outque`, `uart_type`)
- **Side effects:** Writes to FIFO control register to detect 16550 vs 8250; calls `EnablePort()`
- **Calls:** `EnablePort()`
- **Notes:** Prints "UART is a 16550" or "UART is an 8250" to console; initializes queue head/tail/size to 0

### EnablePort
- **Signature:** `void EnablePort(void)`
- **Purpose:** Hook interrupt vector, configure modem control lines, and enable UART/interrupt controller interrupts.
- **Inputs:** None (reads global `uart`, `irq`)
- **Outputs/Return:** None
- **Side effects:** Saves old IRQ vector, installs new ISR, sets DTR/RTS, modifies interrupt mask in PIC (8259)
- **Calls:** `_dos_getvect()`, `_dos_setvect()`, `isr_8250()`, `INPUT()`, `OUTPUT()`, `CLI()`, `STI()`
- **Notes:** Low-level DOS interrupt handling; disables interrupts with `CLI()` during critical sections; sets MCR_DTR and MCR_RTS modem control bits

### ShutdownPort
- **Signature:** `void ShutdownPort(void)`
- **Purpose:** Disable UART interrupts and restore original interrupt vector.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Disables interrupts at UART and PIC; restores original IRQ handler
- **Calls:** `OUTPUT()`, `INPUT()`, `_dos_setvect()`
- **Notes:** Counterpart to `EnablePort()`

### isr_8250
- **Signature:** `void interrupt isr_8250(void)`
- **Purpose:** Hardware interrupt service routine; handles UART receive/transmit/modem-status interrupts.
- **Inputs:** None (reads UART status and data registers via `INPUT()`)
- **Outputs/Return:** None
- **Side effects:** Modifies `inque`, `outque`, `modem_status`, `line_status`; sends/receives bytes to/from hardware
- **Calls:** `INPUT()`, `OUTPUT()`
- **Notes:** Loop-based state machine on interrupt ID; receives all available bytes for 16550 (FIFO), one byte for 8250; transmits up to 16 bytes for 16550, one byte for 8250; sends end-of-interrupt (0x20) to PIC

### read_byte
- **Signature:** `int read_byte(void)`
- **Purpose:** Dequeue one byte from input queue.
- **Inputs:** None (reads `inque`)
- **Outputs/Return:** Byte value (0–255) or −1 if queue empty
- **Side effects:** Advances queue tail, decrements queue size
- **Calls:** `QueSpot()` macro
- **Notes:** No blocking; immediate return if queue empty

### write_byte
- **Signature:** `void write_byte(unsigned char c)`
- **Purpose:** Enqueue one byte to output queue and optionally trigger transmission.
- **Inputs:** `c` – byte to send
- **Outputs/Return:** None
- **Side effects:** Advances queue head, increments queue size; if transmit holding register empty, calls `jump_start()`
- **Calls:** `INPUT()`, `jump_start()`
- **Notes:** No overflow checking; assumes queue has space

### write_bytes
- **Signature:** `void write_bytes(char *buf, int count)`
- **Purpose:** Enqueue multiple bytes, optimized for cases without wraparound.
- **Inputs:** `buf` – source buffer; `count` – number of bytes
- **Outputs/Return:** None
- **Side effects:** May call `write_byte()` in loop (if wraparound) or `memcpy()` (if no wraparound); calls `jump_start()` if TX register empty
- **Calls:** `write_byte()`, `memcpy()`, `INPUT()`, `jump_start()`, `QueSpot()`
- **Notes:** Conditional optimization; checks queue wraparound

### write_buffer
- **Signature:** `void write_buffer(char *buffer, unsigned int count)`
- **Purpose:** Enqueue buffer, dropping existing queue if would overflow.
- **Inputs:** `buffer` – data to send; `count` – size
- **Outputs/Return:** None
- **Side effects:** If `outque.size + count > QUESIZE`, flushes queue (resets head/tail/size); calls `write_bytes()`
- **Calls:** `write_bytes()`
- **Notes:** Destructive overflow handling—existing queued data may be lost

### jump_start
- **Signature:** `void jump_start(void)`
- **Purpose:** Manually prime transmission by writing first byte to transmit holding register.
- **Inputs:** None (reads `outque`)
- **Outputs/Return:** None
- **Side effects:** If queue non-empty, dequeues one byte and writes to UART
- **Calls:** `OUTPUT()`, `QueSpot()`
- **Notes:** Used to initiate transmission when ISR is not immediately available; ISR will continue from there

### ReadSerialPacket
- **Signature:** `boolean ReadSerialPacket(void)`
- **Purpose:** Reassemble frame-based packet from input queue; parse escape sequences.
- **Inputs:** None (reads `inque` and manages global `serialpacket[...]`, `serialpacketlength`, state)
- **Outputs/Return:** `true` if packet complete and ready; `false` if incomplete or buffer overflow
- **Side effects:** Modifies `serialpacket[]`, `serialpacketlength`, `inescape`, `newpacket`; may clear queue on overflow
- **Calls:** `read_byte()`
- **Notes:** Frame format: data bytes, literal `FRAMECHAR` (0x70) escaped as `[0x70, 0x70]`, packet end marked by `FRAMECHAR` not followed by another `FRAMECHAR`; if `inque.size > QUESIZE - 4`, flushes queue and returns false

### WriteSerialPacket
- **Signature:** `void WriteSerialPacket(char *buffer, int len)`
- **Purpose:** Encode and queue outgoing packet with escape sequences.
- **Inputs:** `buffer` – packet data; `len` – packet size
- **Outputs/Return:** None
- **Side effects:** Builds `localbuffer[]` with escaped data, calls `write_buffer()`
- **Calls:** `write_buffer()`
- **Notes:** Escapes `FRAMECHAR` bytes, appends frame terminator (0x70, 0x00); silently drops packets longer than `MAXPACKET`

### talk
- **Signature:** `void talk(void)`
- **Purpose:** Interactive text mode for serial communication testing/debugging.
- **Inputs:** None (reads keyboard via BIOS, reads/writes serial)
- **Outputs/Return:** None (exits on ESC key)
- **Side effects:** Prints to console; reads keyboard and serial, forwards between them
- **Calls:** `_bios_keybrd()`, `write_byte()`, `read_byte()`, `printf()`
- **Notes:** Converts CR (0x0D) to LF (0x0A) on output; loops until ESC detected from keyboard or serial

## Control Flow Notes
**Initialization sequence:** `SetupModemGame()` → `InitPort()` → `EnablePort()` (installs ISR).

**Frame reception:** Main loop calls `ReadSerialPacket()`, which reassembles frames from `inque` populated by `isr_8250()` on each serial interrupt.

**Frame transmission:** Game code calls `WriteSerialPacket()`, which encodes and enqueues to `outque`; `isr_8250()` drains `outque` to hardware, or `write_byte()` calls `jump_start()` to prime transmission.

**Shutdown:** `ShutdownModemGame()` → `ShutdownPort()` (removes ISR, restores vector).

Not tied to explicit update/render loops; interrupt-driven operation independent of game frame timing.

## External Dependencies
- **Includes:** `<conio.h>` (BIOS keyboard), `<dos.h>` (DOS interrupt), `<stdio.h>`, `<stdlib.h>`, `<mem.h>` (memcpy), `<bios.h>` (_bios_keybrd), `memcheck.h` (memory debugging)
- **Local headers:** `rottser.h` (serialdata_t), `_rt_ser.h` (UART register macros, que_t, internal protos), `rt_ser.h` (public API), `rt_def.h` (game constants), `rt_def.h` (define MAXPACKET via MAXPACKETSIZE)
- **Defined elsewhere:** `_dos_getvect()`, `_dos_setvect()` (DOS real-mode interrupt vectors); `inp()`, `outp()` (port I/O); `_disable()`, `_enable()` (CPU interrupt flags); `rottcom` (global config structure); `MAXPACKETSIZE` constant
