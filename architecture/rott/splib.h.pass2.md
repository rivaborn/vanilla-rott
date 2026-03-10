# rott/splib.h — Enhanced Analysis

## Architectural Role
This header defines the input abstraction layer for SpaceWare 3D mouse/force-feedback devices in ROTT. It sits at the hardware-driver boundary, translating DOS-era TSR interrupt protocols into C function calls. The file serves as the sole public interface to SpaceWare hardware; higher-level input systems (notably `rt_spbal.c` for SpaceBall) depend on this as their foundation for 6-DOF device support. This was a premium input option in mid-1990s FPS games.

## Key Cross-References
### Incoming (who depends on this file)
- **`rott/rt_spbal.c`** and **`rott/rt_spbal.h`** (SpaceBall wrapper layer)
  - Functions `CloseSpaceBall` appear in the codebase, indicating a higher-level SpaceBall abstraction wraps these SpaceWare primitives
  - The convenience layer (SpwSimple*) likely feeds into rt_spbal's state management

### Outgoing (what this file depends on)
- **TSR Driver interface** (external DOS-era driver)
  - Defines interrupt codes (`TSR_DRIVER_OPEN`, `TSR_DEVICE_GETFORCE`, etc.) but does NOT implement the interrupt calling mechanism—that's in an implementation file (.c) not shown
  - Packet-based communication with driver (all I/O via SpwPacket union)
- **Memory model abstraction** (`FAR` pointer keyword)
  - Depends on compiler detection (`_MSC_VER`, `__BORLANDC__`) for real-mode vs. protected-mode semantics

## Design Patterns & Rationale

**Packet-Based Protocol**: All driver communication uses a single 128-byte union (`SpwPacket`) with type discrimination via function context. This is characteristic of TSR-era driver APIs where fixed-size buffers and clear request/response contracts were essential. The 128-byte padding signals forward extensibility—anticipating future hardware revisions without protocol changes.

**Convenience Facade**: `SpwSimple*` functions aggregate raw I/O into `SpwRawData`, which combines:
- Current 6-DOF state (translation + rotation vectors)
- Button state with temporal tracking (new/cur/old flags)
- Event type mask (`SpwEventType` enum)

This follows the **facade pattern**: raw polling interfaces (`SpwGetForce`, `SpwGetButton`) are wrapped into a single convenience call. This suggests the library designer expected game code would either use simple convenience functions OR low-level polling, but not mix them.

**Event Type Enumeration**: The `SpwEventType` mask (NO_EVENT, BUTTON_HELD/DOWN/UP, MOTION) shows edge detection awareness—the library attempts to distinguish state changes from continuous state, though this is computed client-side from `SpwButtonRec` (new/cur/old tracking).

## Data Flow Through This File

```
Game Init:
  SpwOpenDriver() → SpwOpenDevice(device_id) → SpwEnableDevice()
  [Populates driver version, device serial via SpwPacket return]

Per-Frame Input Loop (Option 1 - Low-level):
  SpwGetForce(device, packet) → reads SpwForcePacket
  SpwGetButton(device, packet) → reads SpwButtonPacket
  [Game code manually merges into state machine]

Per-Frame Input Loop (Option 2 - Convenience):
  SpwSimpleGet(device, &raw_data) → fills SpwRawData
  [Timestamp & period fields track timing since last event]

Game Shutdown:
  SpwDisableDevice() → SpwCloseDevice() → SpwCloseDriver()
```

**Key observation**: This is a **pull/polling model**, not event-driven. The game calls `SpwSimpleGet()` every frame and checks the `newData` mask to detect state changes. Frame-rate dependent sampling means input granularity is locked to game loop speed (typically 35-70 Hz in 1990s games).

## Learning Notes

**What This Teaches**:
1. **TSR Driver Wrapping (DOS Era)**: This is not a high-level abstraction. It's a thin C wrapper around interrupt-based driver protocol. Real work (interrupt handling, DMA, hardware I/O) happens in the .c implementation and the TSR driver itself.
2. **Hardware Capability Expression**: The `SpwForcePacket` structure (6-DOF + timestamp + period) was considered "advanced" input in 1995—most games only polled keyboard/mouse. This reflects the era's enthusiasm for 3D input peripherals.
3. **Backward Compatibility via Union**: The discriminated union with 128-byte padding shows defensive design—if the driver was updated to include new data fields, old code wouldn't crash (it would see padding).

**What's Idiomatic to This Era**:
- **FAR pointers** for real-mode segmented memory
- **Short return codes** (no error differentiation; client can't tell why it failed)
- **Polling over events** (interrupt handlers and async callbacks were complex in DOS)
- **Manual state tracking** (game code must remember button state across frames to compute edge events)

**Modern Contrast**:
- Modern input APIs (SDL, Raw Input, XInput) use event queues or callbacks
- No memory model macro needed (flat address space)
- Rich error enums (`XINPUT_ERROR_DEVICE_NOT_CONNECTED`, etc.)
- Timestamped events with frame-independence

## Potential Issues

1. **No Error Semantics**: Return codes are `short` with no defined error values. Callers can't distinguish "device not found" from "initialization failed" from "no new data." This forces brittle retry logic in client code.

2. **Manual Button Edge Detection**: `SpwButtonRec` (new/cur/old) requires client code to manually track state across frames to detect button transitions. The `SpwEventType` enum suggests edge detection should be automatic, but the data structure doesn't support it cleanly—mismatches between intent and implementation.

3. **Polling Latency**: Timestamp field in `SpwForcePacket` records driver-side time, but there's no documented synchronization with game clock. If the game loop stalls, input polling stalls, and latency is introduced without visibility into root cause.

4. **Single Device Assumption**: Functions take `device` parameter, but convenience layer `SpwSimpleGet` offers no multi-device aggregation. Games with multiple input devices would need custom multiplexing.
