# rottcom/rottser/port.h — Enhanced Analysis

## Architectural Role

This file provides low-level serial port hardware abstraction for the `rottcom/rottser/` modem/network subsystem. It isolates interrupt-driven UART I/O behind a circular-queue buffer interface, enabling async serial communication without blocking the game loop. The file bridges hardware (8250/16550 UART chips, DOS IRQ vectors) and software (serial protocol handlers like `sermodem.h`, `sercom.h`), allowing the rest of the engine to queue data and forget about timing.

## Key Cross-References

### Incoming (who depends on this file)
- **`rottcom/rottser/sermodem.c`** — Uses `read_byte()`, `write_byte*()` for modem AT command I/O and Hayes protocol implementation
- **`rottcom/rottser/sercom.c`** — Uses `read_byte()`, `write_byte*()` for serial link protocol (framing, checksums, multiplexing)
- Serial port initialization chain likely called from main network setup (entry point not visible in excerpt)

### Outgoing (what this file depends on)
- **DOS/BIOS kernel** — `<dos.h>` provides `interrupt` keyword, interrupt table manipulation, I/O port access
- **UART hardware** — Direct I/O port reads/writes via inline ISRs (I/O port addresses defined in corresponding `.c`)
- **Global state** — Reads/writes `irq`, `uart`, `comport`, `baudrate` (configured elsewhere, likely by `GetUart()` and command-line parsing in `st_cfg.c`)

## Design Patterns & Rationale

**Circular Queue Pattern (power-of-2 sizing)**
- `que_t` uses head/tail pointers and 2048-byte buffer, masking with `(QUESIZE - 1)` for wrap-around
- Avoids modulo overhead on every enqueue/dequeue in interrupt handler
- Double-buffering: `inque` receives serial data, `outque` sends game data; ISR services both

**Separate ISRs for 8250 vs 16550**
- 8250: Single-byte FIFO, slower interrupts, simpler handler
- 16550: 16-byte FIFO, can process bursts without re-interrupting as often
- `Is8250()` and `GetUart()` probe hardware at init time; code selects handler via function pointer or explicit if-branch
- Reflects 1990s hardware reality where both chips coexisted

**Non-blocking API**
- `read_byte()` returns immediately (even if queue empty—likely -1 or error flag)
- Application polls queues rather than blocking on I/O
- Allows game loop to remain responsive during modem negotiation, packet transmission

## Data Flow Through This File

1. **Initialization**: `GetUart()` → detects chip type → `InitPort()` → installs ISR, initializes queues → `jump_start()` enables transmission
2. **Receive path**: Hardware serial byte → ISR (triggered on UART RX interrupt) → appends to `inque.head` → application calls `read_byte()` to drain `inque.tail`
3. **Transmit path**: Application calls `write_byte*()` to append to `outque.head` → ISR detects head != tail → writes to UART I/O port → ISR advances `outque.tail`
4. **Shutdown**: `ShutdownPort()` uninstalls ISR, restores original interrupt vector

## Learning Notes

- **DOS/Real-Mode Interrupt Handling**: The `interrupt` keyword (DOS C dialect) auto-saves registers, calls ISR, and issues IRET. Modern x86 kernels use different mechanisms; understanding this shows how OS-level abstractions have evolved.
- **Hardware Interrupt Servicing**: ISRs must be fast and non-reentrant. The circular queue is chosen because it's lock-free and works without disabling interrupts (critical section is just updating one `head` or `tail` pointer).
- **UART Chip Differences**: 16550 FIFO made a real difference in 9600+ baud serial; 8250 users often saw overflow. This dual-path code reflects empirical platform support.
- **Async I/O Pattern (1990s era)**: Predates async/await; uses ISR + queue buffer instead of callbacks. Modern engines use event-driven I/O or async threads.

## Potential Issues

- **Race Condition Risk**: If `write_byte*()` and ISR both modify `outque.head`/`tail` without atomic instructions, corrupted queue state is possible under unlucky timing. (No mutex/disable-IRQ visible in header.)
- **No Flow Control**: No XON/XOFF or RTS/CTS handling declared; risk of buffer overflow if remote sends data faster than application drains `inque`.
- **Platform-Specific**: DOS/DPMI only. No Windows, Linux, or modern OS support; makes codebase non-portable without reimplementation.
