# rottcom/rottser/sercom.h — Enhanced Analysis

## Architectural Role

This header defines the low-level serial driver interface for Rise of the Triad's networked multiplayer subsystem. It abstracts DOS-era RS-232 hardware interrupts and null-modem/modem communication, sitting between raw serial I/O and higher-level network protocol handlers. The file is central to the `rottcom` (communications) layer, specifically the `rottser` (serial) transport, providing both data transfer and hardware-level diagnostics.

## Key Cross-References

### Incoming (who depends on this file)
From the cross-reference index, `Connect()` is defined in both `rottcom/rottser/sercom.c` and `sercom.h`, indicating it's a public entry point. While the provided cross-reference excerpt does not show all callers, typical callers would be:
- Game initialization code (to establish multiplayer connection)
- Main network loop (to poll for packets via `ReadPacket()`)
- Network packet transmission code (to send state via `WritePacket()`)
- Likely also: modem/protocol negotiation code (sermodem.c appears in context, suggesting serial modem support)

### Outgoing (what this file depends on)
The declarations reveal dependencies on:
- **Hardware interrupt controller** (via `NetISR` callback registration, not visible here)
- **Timing subsystem** (via `StartTime()` / `EndTime()` - likely using DOS timer or CPU cycle counter)
- **Serial hardware I/O ports** (implied by counter names: break, framing error, parity error, overrun)
- No visible C standard library dependencies beyond basic types

## Design Patterns & Rationale

1. **Interrupt-Driven I/O + Polling**: `NetISR()` fires asynchronously on hardware interrupt; `ReadPacket()` is a polling function. This hybrid approach was standard for DOS network drivers—hardware manages buffering in ISR, main loop consumes at its own pace.

2. **Counter-Based Diagnostics**: Seven global error counters rather than logged messages. This reflects memory constraints (no string tables for errors) and the need for lightweight profiling without disk I/O.

3. **Opaque Packet Buffers**: `WritePacket(char *buffer, int len)` takes raw pointers; `ReadPacket()` returns boolean without exposing destination. This suggests packet buffers are managed elsewhere (likely a ring buffer in sercom.c), keeping the header clean.

4. **Timing Instrumentation**: `StartTime()` / `EndTime()` are separately callable, not integrated with packet functions. This allows wrapping arbitrary critical sections for latency profiling—useful when debugging network lag in a DOS environment with no profilers.

## Data Flow Through This File

**Outgoing (player → network):**
1. Game loop calls `WritePacket(buffer, len)` with local state (player position, weapons fired, etc.)
2. Implementation queues in transmit buffer; may trigger `numTxInterrupts` if buffer full
3. Hardware ISR (`NetISR()`) drains transmit buffer to serial port; increments counter

**Incoming (network → game):**
1. Serial hardware receives bytes; generates interrupt
2. `NetISR()` runs (in interrupt context): queues in receive buffer, increments `numRxInterrupts`
3. Main loop calls `ReadPacket()` → checks if complete packet available in buffer
4. If yes, returns `true`; game code reads packet from shared buffer (location unknown from this header)
5. Error counters (`numBreak`, `numFramingError`, etc.) updated by ISR when hardware flags errors

**Diagnostics (developer):**
- `reset_counters()` zeroes all globals before a test run
- `stats()` prints counter values (likely to console)
- Timing: wrap sections with `StartTime()` / `EndTime()` to measure latency

## Learning Notes

- **Era-Specific Hardware Model**: This reflects 1990s DOS-era serial networking. Modern games use TCP/UDP sockets (OS-managed buffers) rather than raw ISR hooks. The manual interrupt handling here would be impossible on modern systems without kernel drivers.

- **Minimalist Error Handling**: No error strings, no exceptions. Errors tracked as counts only. Developers had to correlate counter spikes with game behavior manually.

- **Shared Memory Without Synchronization**: Global counters incremented from both main thread (`ReadPacket`, `WritePacket`) and ISR (`NetISR`). This is a classic race condition, but acceptable in single-core DOS where ISR wouldn't actually preempt (no true multithreading). Modern code would use atomic operations or mutexes.

- **Latency Profiling Primitive**: The separation of `StartTime()` / `EndTime()` from packet I/O suggests developers needed fine-grained profiling. Useful lesson: timing instrumentation can be decoupled from the hot path.

## Potential Issues

1. **Missing Destination Buffer in `ReadPacket()`**: The signature `boolean ReadPacket(void)` doesn't specify *where* the received packet is written. Likely a global buffer in sercom.c, but callers must know the convention (could cause silent buffer overflows).

2. **No Bounds Check on `WritePacket(buffer, int len)`**: Caller is trusted to pass valid buffer and length. No assertion if len exceeds MTU (maximum transmission unit).

3. **Unprotected Global Counters**: Race condition between `NetISR` and main loop incrementing counters. On a multi-core or preemptive OS, this would corrupt statistics. (Not an issue on real DOS single-core hardware, but a red flag if ported to modern systems.)

4. **`Connect()` Return Value Semantics Unclear**: Returns `int` but no error codes defined in this header. Callers must guess what negative/zero values mean.

5. **Timing Granularity Unknown**: `StartTime()` / `EndTime()` give no hint about resolution (CPU cycles? milliseconds? ticks?). Misuse could happen if documentation is missing.
