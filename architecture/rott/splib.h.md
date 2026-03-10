# rott/splib.h

## File Purpose
Header file defining the SpaceWare input device driver interface for the ROTT engine. Provides data structures, function prototypes, and utility wrappers for communicating with 3D mouse/input devices (specifically the SpaceWare Avenger) through TSR interrupt-based driver calls.

## Core Responsibilities
- Define packet structures for driver/device communication (open, force, button data)
- Define TSR interrupt function codes (driver open/close, device control, data retrieval)
- Define enums for device types and input event classifications
- Provide function prototypes for driver lifecycle (open/close)
- Provide function prototypes for device lifecycle (open/close, enable/disable)
- Provide function prototypes for raw data polling (force vectors, button states)
- Provide convenience wrapper functions that aggregate input into a simplified data structure

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| SpwDrvOpenPacket | struct | Driver metadata returned on open (copyright, version, device count) |
| SpwDevOpenPacket | struct | Device metadata returned on open (copyright, serial number) |
| SpwForcePacket | struct | 6-DOF input data: translation (tx/ty/tz) and rotation (rx/ry/rz) vectors |
| SpwButtonPacket | struct | Button input: timestamp, period, button mask |
| SpwCommandPacket | struct | Command packet with validation sentinel (TSRCMD_DATA) |
| SpwPacket | union | Discriminated union holding all packet types (128 bytes padded for expansion) |
| SpwDeviceType | enum | Device type identifier (SPW_AVENGER=1) |
| SpwEventType | enum | Event classification mask (NO_EVENT, BUTTON_HELD/DOWN/UP, MOTION) |
| SpwButtonRec | struct | Button state tracking (new/cur/old) for the convenience layer |
| SpwRawData | struct | Aggregated input data for convenience functions (all axes, button record, event mask) |

## Global / File-Static State
None.

## Key Functions / Methods

### SpwOpenDriver
- Signature: `short SpwOpenDriver(SpwPacket FAR *packet)`
- Purpose: Initialize the SpaceWare driver
- Inputs: Pointer to SpwPacket (caller allocates, driver writes drvOpen packet)
- Outputs/Return: Short status code
- Side effects: Opens TSR driver; populates packet with driver info

### SpwCloseDriver
- Signature: `short SpwCloseDriver(void)`
- Purpose: Shut down the SpaceWare driver
- Outputs/Return: Short status code
- Side effects: Closes TSR driver

### SpwOpenDevice
- Signature: `short SpwOpenDevice(short device, SpwPacket FAR *packet)`
- Purpose: Open a specific input device
- Inputs: Device number, pointer to SpwPacket
- Outputs/Return: Short status code
- Side effects: Populates packet with device info (copyright, serial)

### SpwCloseDevice
- Signature: `short SpwCloseDevice(short device)`
- Purpose: Close a device
- Inputs: Device number
- Outputs/Return: Short status code

### SpwEnableDevice / SpwDisableDevice
- Signature: `short SpwEnableDevice(short device)` / `short SpwDisableDevice(short device)`
- Purpose: Enable/disable data transmission from a device
- Inputs: Device number
- Outputs/Return: Short status code

### SpwGetForce
- Signature: `short SpwGetForce(short device, SpwPacket FAR *packet)`
- Purpose: Poll 6-DOF force/rotation data from device
- Inputs: Device number, pointer to SpwPacket
- Outputs/Return: Short status code; packet contains force data
- Side effects: Updates timestamp and period fields in SpwForcePacket

### SpwGetButton
- Signature: `short SpwGetButton(short device, SpwPacket FAR *packet)`
- Purpose: Poll button state from device
- Inputs: Device number, pointer to SpwPacket
- Outputs/Return: Short status code; packet contains button data

### SpwSimpleGet
- Signature: `short SpwSimpleGet(short devNum, SpwRawData FAR *splayer)`
- Purpose: Convenience wrapper that polls both force and button data, aggregating into a single data structure
- Inputs: Device number, pointer to SpwRawData
- Outputs/Return: Short status code
- Side effects: Populates all fields of SpwRawData in one call

### SpwSimpleOpen / SpwSimpleClose
- Signature: `short SpwSimpleOpen(short devNum)` / `short SpwSimpleClose(short devNum)`
- Purpose: Convenience wrappers for device open/close
- Inputs: Device number
- Outputs/Return: Short status code

## Control Flow Notes
Expected usage flow:
1. **Init phase**: `SpwOpenDriver()` → `SpwOpenDevice()` → `SpwEnableDevice()`
2. **Update phase**: Per-frame call to `SpwGetForce()` / `SpwGetButton()` (or `SpwSimpleGet()` for convenience)
3. **Shutdown phase**: `SpwDisableDevice()` → `SpwCloseDevice()` → `SpwCloseDriver()`

The `SpwSimple*` functions are provided for simplified polling that aggregates all input in one call.

## External Dependencies
- **Conditional compiler directives**: `__cplusplus` (C++ extern guard), `_MSC_VER`, `__BORLANDC__` (REALMODE memory model detection)
- **FAR keyword**: Conditionally defined for real-mode vs. protected-mode compilation
- **Documentation reference**: "SpReadme.doc" (external user guide)
- No external library dependencies; purely C header definitions

## Notes
- TSR (Terminate and Stay Resident) interrupt function codes are defined but actual interrupt mechanics are implemented elsewhere
- The `FAR` pointer type indicates real-mode x86 compatibility (legacy DOS/early Windows era)
- Packet-based communication design allows for versioning and future expansion (128-byte union padding)
