# rott/spw_int.h

## File Purpose
Header file for SpaceTec spacemouse/input device integration. Defines the packet structure used to communicate input data (6DOF motion and button states) from SpaceTec hardware to the game engine.

## Core Responsibilities
- Define the `Spw_IntPacket` structure for spacemouse data
- Declare button ID constants for spacemouse buttons
- Declare initialization, polling, and cleanup functions for the SpaceTec device driver
- Provide a data format contract between the hardware driver (implementation) and consumers

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Spw_IntPacket` | struct | Contains one complete input sample from spacemouse: 6DOF translation/rotation axes, button states, checksum, and timing data |

## Global / File-Static State
None.

## Key Functions / Methods

### SP_Open
- Signature: `short SP_Open(void)`
- Purpose: Initialize and open connection to SpaceTec input device
- Inputs: None
- Outputs/Return: `short` (likely status code; 0 = success or error flag)
- Side effects: Opens hardware device, allocates driver resources
- Calls: Not visible in header
- Notes: Must be called before `SP_Get()` or `SP_Get_Btn()`; likely paired with `SP_Close()`

### SP_Get
- Signature: `void SP_Get(Spw_IntPacket* sp)`
- Purpose: Poll the spacemouse and populate packet with latest input state
- Inputs: Pointer to `Spw_IntPacket` buffer to fill
- Outputs/Return: Data written to `*sp`; no return value
- Side effects: Reads from hardware device
- Calls: Not visible in header
- Notes: Caller must allocate packet structure

### SP_Get_Btn
- Signature: `void SP_Get_Btn(Spw_IntPacket* sp)`
- Purpose: Update button state in the packet (specialized button polling)
- Inputs: Pointer to `Spw_IntPacket` to update
- Outputs/Return: Button fields modified in `*sp`; no return value
- Side effects: Reads button state from hardware
- Calls: Not visible in header
- Notes: May be called separately or as variant of `SP_Get()`

### SP_Close
- Signature: `void SP_Close(void)`
- Purpose: Close connection and release resources for SpaceTec device
- Inputs: None
- Outputs/Return: None
- Side effects: Releases hardware device handle and driver resources
- Calls: Not visible in header
- Notes: Should be called during game shutdown

## Control Flow Notes
Inferred initialization/shutdown pattern: `SP_Open()` → repeated `SP_Get()` or `SP_Get_Btn()` calls per frame → `SP_Close()` on exit. Likely called from input polling subsystem or controller initialization code.

## External Dependencies
- No includes visible (header-only declarations)
- SpaceTec hardware driver implementation (defined elsewhere)
