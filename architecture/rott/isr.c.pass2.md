# rott/isr.c — Enhanced Analysis

## Architectural Role
This file is the **hardware abstraction layer** for the game's two most critical real-time subsystems: game timing and raw keyboard input. Rather than installing ISRs directly, it delegates to a task-based scheduler (`task_man.h`) for timer callbacks, making timer logic more testable and decoupled from interrupt context. The keyboard subsystem remains ISR-driven (required for responsiveness) and feeds a circular queue that higher-level input handlers consume. Together, these form the heartbeat and nerve system of the engine.

## Key Cross-References

### Incoming (who depends on this file)
- **`rt_main.c`, game loop**: Polls `ticcount` for frame timing and frame-skipping decisions
- **`rt_in.h`, input system**: Reads from `KeyboardQueue[]`, `Keystate[]`, `LastScan`, and LED control functions
- **`rt_game.c`, game logic**: Calls `I_Delay()` for UI waits (pause screens, menus)
- **`rt_playr.c`, player control**: Reads `Keystate[]` for continuous key-down checks (movement, strafing)
- **`rt_menu.c`, menu system**: Uses `I_Delay()` and checks pause flag (`PausePressed`)
- **Task manager (`task_man.h`)**: Receives timer task callbacks via `ISR_Timer()` at scheduled intervals

### Outgoing (what this file depends on)
- **`task_man.h`**: `TS_ScheduleTask()`, `TS_Dispatch()`, `TS_Terminate()`, `TS_Shutdown()` — delegates timer ISR to task scheduler
- **`rt_in.h`**: Reads `LastScan` (external input state), calls `IN_ClearKeysDown()` in `I_Delay()`
- **`rt_util.h`**: Calls `Error()` for fatal failures (invalid timer speed)
- **DOS/BIOS** (via macros): `OUTP`, `inp`, `_disable()`, `_enable()` for hardware I/O and interrupt masking; direct writes to BIOS keyboard buffer (0x41c, 0x41a)
- **`profile.h`, `develop.h`**: Conditional development profiling (mostly compiled out)

## Design Patterns & Rationale

**Task-Based Scheduler (vs. Direct ISR):**
The timer no longer directly modifies game state from interrupt context. Instead, it schedules a task (`ISR_Timer`) that the main loop dispatches. This reduces interrupt-context complexity and avoids race conditions. Legacy code shows direct ISR hooking was attempted; the current design is cleaner.

**Circular Queue for Keyboard Events:**
`KeyboardQueue[]` with `Keyhead`/`Keytail` decouples hardware interrupts (producer) from game logic (consumer), allowing variable frame rates without losing input.

**Interrupt Safety Through Volatile:**
All shared state between ISR and main code is `volatile` (e.g., `ticcount`, `Keystate[]`, `ExtendedKeyFlag`), preventing compiler optimizations that would break interrupt semantics.

**Hardware Handshake for Keyboard Commands:**
`I_SendKeyboardData()` implements a retry loop with interrupt-driven flags (`KBFlags`, set by keyboard ISR), rather than polling. This is efficient for slow PS/2 controller communication.

**Extended Key Prefix Tracking:**
The 0xE0 prefix (and 0xE1 for pause) requires state across multiple interrupts; `ExtendedKeyFlag` handles this in the ISR, not main code.

## Data Flow Through This File

**Timer Flow:**
1. **Hardware** (PIT) fires ~35 Hz → `I_TimerISR()` → `OUTP(0x20, 0x20)` (PIC ack)
2. Task scheduler increments `ticcount` (via `ISR_Timer()` callback)
3. **Main loop** polls `ticcount` for frame timing
4. `I_Delay()` spins on `ticcount` + offset for UI waits

**Keyboard Flow:**
1. **Hardware** (PS/2 controller) fires on key press/release → `I_KeyboardISR()`
2. ISR reads scan code from port 0x60, manages `ExtendedKeyFlag`
3. Scan code → `KeyboardQueue[]`, `Keystate[]` updated
4. **Main input handler** (rt_in.c) dequeues and converts to logical keys

**Initialization Sequence:**
- `I_StartupTimer()`: Sync CMOS time, schedule timer task, reset `ticcount` to 0
- `I_StartupKeyboard()`: Save old ISR vector, hook new ISR, clear LED state
- Both install interrupt handlers via `_dos_setvect()`

## Learning Notes

**DOS-Era Hardware Patterns:**
- Direct port I/O (`inp`, `outp`) and BIOS data area manipulation (0x41c for BIOS keyboard buffer) are now encapsulated in OSes, but here you see raw hardware access
- CMOS RTC reads with BCD decoding show era-specific firmware quirks
- Interrupt handler chaining (saving old vectors) is how modular DOS programs coexist

**Contrast to Modern Engines:**
- Modern engines use OS-provided input events (Windows `WM_KEYDOWN`, SDL event loop) instead of direct ISR hooking
- Timing via `gettimeofday()` or OS sleep primitives, not PIT reprogramming
- No need for interrupt masking (`_disable()/_enable()`) in user-space; OS handles that

**Task Scheduler Integration:**
This file shows a clever pattern: **hardware timing is abstracted through a scheduler**. The main loop calls `TS_Dispatch()`, which triggers timer callbacks. This is similar to how modern event-driven systems (reactor pattern, libuv) handle timers—decouple hardware from logic.

**Keyboard State Management:**
The combination of:
- Per-scan-code state (`Keystate[]`)
- Event queue (`KeyboardQueue[]`)
- Special flags (extended keys, pause key)

...mirrors modern game input systems (e.g., SFML, SDL) but handled manually in interrupt context here.

## Potential Issues

1. **Race Condition in `I_Delay()`**: The loop `while (ticcount < time+delay)` reads a global volatile written by ISR. If `ticcount` overflows (after ~49 days at 35 Hz), wrap-around logic is missing. However, realistic session length is short enough that this doesn't matter in practice.

2. **BIOS Buffer Direct Write**: `I_ShutdownKeyboard()` writes directly to BIOS keyboard buffer addresses (0x41c, 0x41a). This is fragile if BIOS layout changes (unlikely on fixed hardware) and unsafe on modern systems.

3. **Keyboard Handshake Timeout**: `I_SendKeyboardData()` spins on `KBFlags` for up to 0x1a00 iterations. On very slow hardware or under interrupt contention, this could time out silently (setting `kb_error` flag without halting). LED commands failing silently might confuse users.

4. **No Guard Against Multiple Calls**: `I_StartupTimer()` and `I_StartupKeyboard()` check `TimerStarted` and `KeyboardStarted` flags, but concurrent calls (or shutdown→restart) could cause double-installation of ISR vectors.
