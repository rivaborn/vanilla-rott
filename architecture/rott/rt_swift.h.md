# rott/rt_swift.h

## File Purpose
Public header declaring the interface for SWIFT haptic feedback device support. Provides initialization, device detection, input polling, and tactile feedback control for specialized 3D input devices in Rise of the Triad.

## Core Responsibilities
- Initialize and detect presence of SWIFT device extensions
- Terminate and free SWIFT-related resources
- Query attached device type and static configuration
- Poll 6DOF input status (position, orientation, buttons) each frame
- Generate tactile feedback (motor on/off cycling)
- Read dynamic device state and capabilities

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SWIFT_3DStatus` | struct | Interactive input state: 3D position, pitch/roll/yaw orientation, button state |
| `SWIFT_StaticData` | struct | Static device metadata: type, version, coordinate descriptor |

## Global / File-Static State
None.

## Key Functions / Methods

### SWIFT_Initialize
- **Signature:** `int SWIFT_Initialize(void)`
- **Purpose:** Test for presence of SWIFT extensions and attached device
- **Inputs:** None
- **Outputs/Return:** 1 (TRUE) if SWIFT features available; 0 otherwise
- **Side effects:** Allocates resources for SWIFT support if successful
- **Calls:** None visible in header
- **Notes:** Must be paired with `SWIFT_Terminate()` on success

### SWIFT_Terminate
- **Signature:** `void SWIFT_Terminate(void)`
- **Purpose:** Free resources allocated by SWIFT_Initialize
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deallocates SWIFT resources; safe to call even if SWIFT_Initialize was not called or failed
- **Calls:** None visible in header
- **Notes:** Should always be called after successful SWIFT_Initialize

### SWIFT_GetAttachedDevice
- **Signature:** `int SWIFT_GetAttachedDevice(void)`
- **Purpose:** Identify connected SWIFT device type
- **Inputs:** None
- **Outputs/Return:** Device-type code
- **Side effects:** Reads hardware state
- **Calls:** None visible in header
- **Notes:** Returns 0 if no device attached

### SWIFT_GetStaticDeviceInfo
- **Signature:** `int SWIFT_GetStaticDeviceInfo(SWIFT_StaticData far *psd)`
- **Purpose:** Retrieve immutable device characteristics (type, version, coordinate system)
- **Inputs:** Pointer to SWIFT_StaticData struct
- **Outputs/Return:** Populates psd; return value not documented
- **Side effects:** Reads device firmware metadata
- **Calls:** None visible in header
- **Notes:** Far pointer indicates real-mode/DOS memory model; typically called once at startup

### SWIFT_Get3DStatus
- **Signature:** `void SWIFT_Get3DStatus(SWIFT_3DStatus far *pstat)`
- **Purpose:** Poll current input state (6DOF position/orientation, buttons)
- **Inputs:** Pointer to SWIFT_3DStatus struct
- **Outputs/Return:** Populates pstat with x, y, z, pitch, roll, yaw, buttons
- **Side effects:** Reads device hardware state each frame
- **Calls:** None visible in header
- **Notes:** Expected to be called per-frame during input polling; far pointer for real-mode memory

### SWIFT_TactileFeedback
- **Signature:** `void SWIFT_TactileFeedback(int d, int on, int off)`
- **Purpose:** Generate haptic feedback via motor on/off cycling
- **Inputs:** 
  - `d`: duration of tactile burst (milliseconds)
  - `on`: motor on-time per cycle (milliseconds)
  - `off`: motor off-time per cycle (milliseconds)
- **Outputs/Return:** None
- **Side effects:** Controls haptic device motor; audible/tactile output
- **Calls:** None visible in header
- **Notes:** Allows tunable feedback intensity and frequency through PWM-like pattern

### SWIFT_GetDynamicDeviceData
- **Signature:** `unsigned SWIFT_GetDynamicDeviceData(void)`
- **Purpose:** Retrieve runtime device state flags and capabilities
- **Inputs:** None
- **Outputs/Return:** Bit-packed device data word (specific flags defined as `SDD_*` elsewhere)
- **Side effects:** Reads device hardware state
- **Calls:** None visible in header
- **Notes:** Complements `SWIFT_Get3DStatus` for non-positional state

## Control Flow Notes
This module fits into the **input polling** phase of the game loop. SWIFT_Initialize is called during engine startup; SWIFT_Get3DStatus and SWIFT_GetDynamicDeviceData are polled each frame alongside keyboard, mouse, and joystick input (per rt_playr.h). SWIFT_TactileFeedback is called on-demand when game events require haptic response (e.g., weapon fire, impact). SWIFT_Terminate is invoked at shutdown.

## External Dependencies
- **Included:** `rt_playr.h` — provides `SWIFT_3DStatus` and `SWIFT_StaticData` typedef definitions
- **Memory model:** Far pointers indicate real-mode DOS protected mode or segmented memory architecture
