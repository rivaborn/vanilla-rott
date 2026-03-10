# rott/vrio.h

## File Purpose
API documentation header for Virtual Reality input device integration in ROTT. Defines interrupt-based communication protocol (INT 0x33) for reading VR controller input and sending haptic feedback to the VR device.

## Core Responsibilities
- Document interrupt handler 0x30 (GetVRInput) for reading VR controller state and mouse input
- Document interrupt handler 0x31 (VRFeedback) for sending haptic feedback to the VR device
- Define button bit positions for 16 different VR controller inputs
- Specify angle normalization convention (0..2047 range, no negative angles)
- Document register conventions for passing input parameters and receiving output

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
This is a documentation-only header file. No function implementations. The file documents two interrupt-based API endpoints:

### GetVRInput (INT 0x33 AX = 0x30)
- **Purpose:** Read current VR controller button state and mouse movement
- **Inputs:** 
  - BX: current player angle [0..2048)
  - CX: current player tilt angle [0..2048) (up: 0–171, down: 1876–2047)
- **Outputs/Return:**
  - BX: button status bitmask (16 buttons, bits 0–15)
  - CX: mouse X mickeys
  - DX: mouse Y mickeys
- **Notes:** All angles normalized to range [0..2048); no negative angles permitted

### VRFeedback (INT 0x33 AX = 0x31)
- **Purpose:** Send haptic feedback command to VR device
- **Inputs:**
  - BX: control flag (0=stop, 1=start feedback)
  - CX: weapon type (0=gun, 1=missile)
- **Notes:** Used to synchronize device feedback with gameplay events

## Control Flow Notes
Used during the game loop's input phase to poll VR device state and during event handling to trigger haptic feedback responses.

## External Dependencies
None—pure API documentation.
