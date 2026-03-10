# rottcom/rottser/port.c

## File Purpose
Low-level serial port driver for DOS-based system that manages UART (8250/16550) hardware initialization, interrupt-driven buffered I/O, and modem control signals. Provides serial communication backend for multi-player game networking.

## Core Responsibilities
- Detect UART type (8250 vs 16550) and IRQ via BIOS system data
- Initialize UART with baud rate, divisor, and control line settings (DTR, RTS)
- Hook IRQ vector and manage interrupt service routine
- Maintain circular input/output queues for buffered byte transfer
- Handle UART interrupt events (RX, TX, modem status, line errors)
- Provide high-level read/write API with buffer overflow protection

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `que_t` | struct | Circular queue with head, tail, size, and QUESIZE data buffer for RX/TX |
| `REGS` | union | x86 CPU registers (from dos.h) for BIOS int86/int86x calls |
| `SREGS` | struct | x86 segment registers for extended BIOS calls |
| `uart_type` (enum) | enum | UART_8250 or UART_16550 variant selector |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `inque`, `outque` | que_t | global | RX and TX circular buffers |
| `irq` | int | global | Detected IRQ number (-1 if auto-detect) |
| `uart` | int | global | Base I/O port address (-1 if auto-detect) |
| `comport` | int | global | COM port number (1–4) |
| `baudrate` | long | global | Configured baud rate (default 9600) |
| `uart_type` | enum | file-static | Detected UART variant (8250 or 16550) |
| `modem_status`, `line_status` | int | global | Last sampled hardware status registers |
| `oldirqvect` | void interrupt (...) | file-static | Saved IRQ vector for restoration |
| `irqintnum` | int | file-static | Calculated interrupt number (irq + 8) |
| `regs`, `sregs` | REGS, SREGS | file-static | x86 register staging for BIOS calls |

## Key Functions / Methods

### GetUart
- **Signature:** `void GetUart(void)`
- **Purpose:** Auto-detect UART base port and IRQ by checking system configuration. Falls back to ISA defaults if MCA (Micro Channel) not detected.
- **Inputs:** Global `comport` (1–4); system BIOS data
- **Outputs/Return:** Updates global `uart` and `irq`; prints discovery message
- **Side effects:** x86 BIOS int86x call (interrupt 0x15 with ah=0xc0); reads far memory at ES:BX
- **Calls:** `int86x()`
- **Notes:** Only updates `irq` and `uart` if currently -1 (not manually set); supports ISA (standard) and MCA (PS/2) I/O mappings

### InitPort
- **Signature:** `void InitPort(void)`
- **Purpose:** Complete UART initialization: detect 16550 vs 8250, set baud rate divisor, configure modem control lines (DTR, RTS), install ISR, enable UART and PIC interrupts.
- **Inputs:** Global `uart`, `irq`, `baudrate`, `comport`
- **Outputs/Return:** None (modifies hardware state and global state)
- **Side effects:** Direct UART register writes; IRQ vector hook installation; interrupt controller (PIC) reprogramming; queues zeroed; disables then re-enables CPU interrupts (CLI/STI)
- **Calls:** `INPUT()`, `OUTPUT()` (hardware I/O), `getvect()`, `setvect()` (vector management), `isr_8250()` (ISR address)
- **Notes:** Caps 8250 baud rate at 57,600 baud; 16550 FIFO set to 4-byte trigger; commented code suggests 16550 was conditionally unsupported in early release; divisor = CLOCK_FREQUENCY / (16 × baudrate); modem control includes OUT2 and loopback disable

### ShutdownPort
- **Signature:** `void ShutdownPort(void)`
- **Purpose:** Disable UART interrupts, restore original IRQ vector, clear modem control.
- **Inputs:** None (reads global `uart`, `irq`, `irqintnum`, `oldirqvect`)
- **Outputs/Return:** None
- **Side effects:** Writes to UART control registers; PIC reprogramming; ISR vector restoration
- **Calls:** `INPUT()`, `OUTPUT()`, `setvect()`
- **Notes:** Inverse of `InitPort()`; called on cleanup/exit

### read_byte
- **Signature:** `int read_byte(void)`
- **Purpose:** Dequeue one byte from RX buffer (inque), blocking semantic responsibility on caller.
- **Inputs:** None (reads global `inque`)
- **Outputs/Return:** Dequeued byte (0–255) or -1 if queue empty
- **Side effects:** Modifies `inque.tail` and `inque.size`; wraps tail via `QueSpot()` macro
- **Calls:** None
- **Notes:** Non-blocking; caller must check for -1; no bounds check (assumes queue not corrupted)

### write_byte
- **Signature:** `void write_byte(unsigned char c)`
- **Purpose:** Enqueue one byte to TX buffer and prime transmit interrupt if hardware ready.
- **Inputs:** `c` (byte to transmit)
- **Outputs/Return:** None
- **Side effects:** Updates `outque.head` and `outque.size`; may call `jump_start()` if LINE_STATUS_REGISTER bit 0x40 (THRE) is set
- **Calls:** `INPUT()`, `jump_start()`
- **Notes:** Does not check for queue overflow; wraps head via `QueSpot()`

### write_bytes
- **Signature:** `void write_bytes(char *buf, int count)`
- **Purpose:** Batch-enqueue bytes, with optimized path for non-wrapping vs. wrapping cases.
- **Inputs:** `buf` (byte array), `count` (number of bytes)
- **Outputs/Return:** None
- **Side effects:** Modifies `outque`; may call `memcpy()` or `write_byte()` in loop; calls `jump_start()` if THRE ready
- **Calls:** `memcpy()`, `write_byte()`, `INPUT()`, `jump_start()`
- **Notes:** Avoids memcpy if write would wrap around buffer edge; does not check overflow

### write_buffer
- **Signature:** `void write_buffer(char *buffer, unsigned int count)`
- **Purpose:** High-level buffered write with overflow protection: discard queue if enqueue would exceed QUESIZE.
- **Inputs:** `buffer` (byte array), `count` (count of bytes)
- **Outputs/Return:** None
- **Side effects:** May zero `outque` and increment `writeBufferOverruns` if overflow detected; calls `write_bytes()`
- **Calls:** `write_bytes()`
- **Notes:** Protective wrapper; prevents buffer overflow by sacrificing previous unacknowledged data

### isr_8250
- **Signature:** `void interrupt isr_8250(void)`
- **Purpose:** Interrupt service routine for UART; handles RX, TX, modem status, and line status events.
- **Inputs:** None (reads UART registers and queues)
- **Outputs/Return:** None
- **Side effects:** Modifies `inque` and `outque`; reads/writes UART registers; updates `modem_status` and `line_status`; increments statistics counters
- **Calls:** `INPUT()`, `OUTPUT()`, loop over interrupt ID register until IIR_NO_INTERRUPT
- **Notes:** Handles both 8250 (1 byte RX/TX per interrupt) and 16550 (up to 16 bytes RX/TX); acknowledges PIC with OUTPUT(0x20, 0x20); line error tracking: overrun, parity, framing, break; services interrupts in priority order (loop continues until no more interrupts pending)

### jump_start
- **Signature:** `void jump_start(void)`
- **Purpose:** Prime transmit by writing first byte directly to UART, triggering TX interrupt chain.
- **Inputs:** None (reads global `outque`)
- **Outputs/Return:** None
- **Side effects:** Writes one byte to UART + TRANSMIT_HOLDING_REGISTER; updates `outque.tail` and `outque.size`
- **Calls:** `OUTPUT()`
- **Notes:** Called from `write_byte()` and `write_bytes()` when THRE ready; bootstraps TX interrupt sequence

## Control Flow Notes
**Initialization Phase:** `GetUart()` → `InitPort()` establish port configuration and hook IRQ vector.  
**Runtime Phase:** Application calls `write_buffer()` / `write_bytes()` / `write_byte()` to send; calls `read_byte()` in a polling loop to receive. Hardware interrupts trigger `isr_8250()` asynchronously.  
**Shutdown Phase:** `ShutdownPort()` disables interrupts, clears modem control, restores original IRQ vector.

## External Dependencies
- **Included headers:** `<conio.h>`, `<dos.h>` (x86/DOS primitives), `<stdio.h>`, `<stdlib.h>`, `<mem.h>`
- **Local headers:** `global.h` (CLOCK_FREQUENCY, INPUT/OUTPUT/CLI/STI macros), `port.h` (que_t, extern queues and port config), `serial.h` (UART register offsets and constants), `sercom.h` (statistics externs), `sersetup.h` (shutdown extern)
- **Defined elsewhere:** `getvect()`, `setvect()` (DOS vector table), `inp()`, `outp()`, `int86x()` (DOS I/O), `disable()`, `enable()` (CPU interrupt control), `memcpy()` (std lib); statistics counters (`numBreak`, `numFramingError`, etc.), `writeBufferOverruns` in sercom module
