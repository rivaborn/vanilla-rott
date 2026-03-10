# rott/rt_swift.c

## File Purpose
Provides SWIFT device abstraction and control for Cyberman 3D input devices in ROTT. Wraps DPMI real-mode interrupt calls and DOS memory management to communicate with SWIFT extensions via the mouse driver (INT 0x33).

## Core Responsibilities
- Initialize/terminate SWIFT device detection and resource management
- Query attached SWIFT device type and static/dynamic capabilities
- Generate tactile feedback output to Cyberman device
- Manage DOS real-mode memory buffers for device communication
- Execute DPMI real-mode interrupts for device I/O

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SWIFT_StaticData` | struct | Device static information (definition in included header, not provided) |
| `SWIFT_3DStatus` | struct | Current input state of 3D device (definition in included header, not provided) |
| `struct rminfo` | struct | DPMI real-mode interrupt info block; holds register state (ax, bx, cx, dx, etc.) and flags |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `fActive` | int | static | TRUE if SWIFT extensions active and device initialized; FALSE otherwise |
| `nAttached` | int | static | Device type code: SWIFT_DEV_NONE or SWIFT_DEV_CYBERMAN |
| `regs` | union REGS | static | CPU register state for DPMI calls; used to pass/receive register values |
| `sregs` | struct SREGS | static | Segment registers (ES, DS, FS, GS) for DPMI real-mode interrupts |
| `selector` | short | static | Protected-mode selector of allocated DOS memory block |
| `segment` | short | static | Real-mode segment address of DOS memory block |
| `pdosmem` | void far * | static | Protected-mode pointer to DOS real-mode buffer |
| `RMI` | struct rminfo | static | Real-mode interrupt structure; populated before/after DPMI INT 0x31 calls |

## Key Functions / Methods

### SWIFT_Initialize
- **Signature:** `int SWIFT_Initialize(void)`
- **Purpose:** Test for SWIFT device presence and allocate resources if found.
- **Inputs:** None.
- **Outputs/Return:** 1 (TRUE) if SWIFT features and device available; 0 (FALSE) otherwise.
- **Side effects:** Sets `fActive`, `nAttached`, allocates DOS memory via `allocDOS()`, calls `SoftError()` in DEBUG builds.
- **Calls:** `_dos_getvect()`, `int386()`, `allocDOS()`, `SWIFT_GetStaticDeviceInfo()`, `freeDOS()`, `SoftError()`.
- **Notes:** Caller must invoke `SWIFT_Terminate()` if this returns TRUE. Checks for mouse driver first; resets mouse; confirms pointing device exists before attempting SWIFT detection. On failure, frees any allocated DOS memory.

### SWIFT_Terminate
- **Signature:** `void SWIFT_Terminate(void)`
- **Purpose:** Free resources and disable SWIFT module.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `freeDOS()` if DOS memory allocated; sets `fActive = 0`.
- **Calls:** `freeDOS()`, `SoftError()`.
- **Notes:** Safe to call even if `SWIFT_Initialize()` was never called or failed.

### SWIFT_GetAttachedDevice
- **Signature:** `int SWIFT_GetAttachedDevice(void)`
- **Purpose:** Query device type of attached SWIFT device.
- **Inputs:** None.
- **Outputs/Return:** Device type code (SWIFT_DEV_NONE, SWIFT_DEV_CYBERMAN, or unknown type).
- **Side effects:** None.
- **Calls:** None (returns static `nAttached`).
- **Notes:** Returns SWIFT_DEV_NONE if no device attached or module not initialized.

### SWIFT_GetStaticDeviceInfo
- **Signature:** `int SWIFT_GetStaticDeviceInfo(SWIFT_StaticData far *psd)`
- **Purpose:** Read static device information via SWIFT INT 0x33 command 0x53C1.
- **Inputs:** `psd` — pointer to caller's SWIFT_StaticData buffer.
- **Outputs/Return:** 1 (success) if RMI.ax == 1 after interrupt; 0 otherwise. Populates `*psd`.
- **Side effects:** Writes to DOS real-mode buffer; modifies RMI registers.
- **Calls:** `memset()`, `MouseInt()`.
- **Notes:** Copies device data from DOS buffer to caller's buffer. Caller must ensure `psd` is valid; assumes DOS buffer allocated.

### SWIFT_Get3DStatus
- **Signature:** `void SWIFT_Get3DStatus(SWIFT_3DStatus far *pstat)`
- **Purpose:** Read current 3D input state (position, buttons, etc.) via SWIFT INT 0x33 command 0x5301.
- **Inputs:** `pstat` — pointer to caller's SWIFT_3DStatus buffer.
- **Outputs/Return:** None. Populates `*pstat` from DOS buffer.
- **Side effects:** Writes to DOS real-mode buffer; modifies RMI registers.
- **Calls:** `memset()`, `MouseInt()`.
- **Notes:** Meant to be polled each frame or input loop. Caller must ensure `pstat` is valid; asserts `fActive` in DEBUG build.

### SWIFT_TactileFeedback
- **Signature:** `void SWIFT_TactileFeedback(int d, int on, int off)`
- **Purpose:** Generate haptic/tactile feedback pulse to Cyberman device.
- **Inputs:** `d` — duration in milliseconds; `on` — motor on-time per cycle (ms); `off` — motor off-time per cycle (ms).
- **Outputs/Return:** None.
- **Side effects:** Modifies RMI registers; issues SWIFT INT 0x33 command 0x5330.
- **Calls:** `memset()`, `MouseInt()`, `SoftError()`.
- **Notes:** Quantizes inputs (duration /40 ms, on/off /5 ms). Performs bit shift on `on` and `off` values for packed register format.

### SWIFT_GetDynamicDeviceData
- **Signature:** `unsigned SWIFT_GetDynamicDeviceData(void)`
- **Purpose:** Query dynamic device data (power status, etc.) via SWIFT INT 0x33 command 0x53C2.
- **Inputs:** None.
- **Outputs/Return:** Data word from RMI.ax (e.g., SDD_EXTERNAL_POWER_CONNECTED flags).
- **Side effects:** Modifies RMI registers.
- **Calls:** `memset()`, `MouseInt()`.
- **Notes:** Returns unsigned word; caller interprets flags (SDD_* constants).

### MouseInt
- **Signature:** `void MouseInt(struct rminfo *prmi)`
- **Purpose:** Execute a real-mode INT 0x33 (mouse) via DPMI INT 0x31 (simulate real-mode interrupt).
- **Inputs:** `prmi` — pointer to real-mode interrupt info structure (filled by caller with registers and command).
- **Outputs/Return:** None. RMI structure modified in-place with return values.
- **Side effects:** Performs DPMI call; updates register state in `regs` and `sregs`.
- **Calls:** `memset()`, `int386x()`.
- **Notes:** Low-level bridge between protected mode and real-mode mouse driver. Caller must populate RMI before call.

### allocDOS
- **Signature:** `void far *allocDOS(unsigned nbytes, short *pseg, short *psel)`
- **Purpose:** Allocate real-mode DOS memory block via DPMI function 0x0100.
- **Inputs:** `nbytes` — size in bytes; `pseg`, `psel` — output pointers for segment and selector.
- **Outputs/Return:** Protected-mode far pointer to allocated block; NULL on failure. Sets `*pseg` and `*psel` on success.
- **Side effects:** Updates global `regs` and `sregs`; performs DPMI call.
- **Calls:** `segread()`, `int386()`.
- **Notes:** Converts byte count to paragraphs (16-byte units). Protected-mode pointer derived by shifting segment left 4 bits (Rational extender memory model). Caller must check return; use selector for later freeDOS() call.

### freeDOS
- **Signature:** `void freeDOS(short sel)`
- **Purpose:** Release real-mode DOS memory block via DPMI function 0x0101.
- **Inputs:** `sel` — protected-mode selector of block to free.
- **Outputs/Return:** None.
- **Side effects:** Updates `regs`; performs DPMI call.
- **Calls:** `int386()`.
- **Notes:** Complements allocDOS(). Selector must be valid; no error checking in this function.

## Control Flow Notes
Initialization is tied to game startup (likely during engine init phase). `SWIFT_Initialize()` checks device availability and allocates resources. During input polling (per-frame or per-tic), code calls `SWIFT_Get3DStatus()` to read device state. `SWIFT_TactileFeedback()` may be invoked on demand to generate haptic feedback. Shutdown calls `SWIFT_Terminate()` to free DOS memory. The SWIFT layer is optional; if initialization fails, the game continues without 3D input device support.

## External Dependencies
- **Notable includes:** `<dos.h>` (DOS interrupt support), `"rt_def.h"` (engine definitions), `"rt_swift.h"` (public API), `"_rt_swft.h"` (private definitions), `"memcheck.h"` (debug memory tracker).
- **External symbols used (defined elsewhere):**
  - `_dos_getvect()`, `int386()`, `int386x()`, `segread()` — DOS/DPMI interrupt functions (from DOS extender runtime).
  - `SoftError()` — debug error logging function (RT engine).
  - `memset()` — standard C library.
  - `allocDOS()`, `freeDOS()` — defined in this file (static); not exported.
