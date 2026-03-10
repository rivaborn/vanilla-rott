# rott/isr.h

## File Purpose
Header file declaring interrupt service routines (ISRs) for keyboard input and timer management in a DOS/retro environment. Provides low-level hardware interrupt handling, keyboard state tracking, and frame timing via ticcount.

## Core Responsibilities
- Manage keyboard and timer interrupt service routine initialization/shutdown
- Provide circular queue for keyboard input buffering (KeyboardQueue)
- Track real-time keyboard state and detect key changes
- Maintain frame timing counter (ticcount, typically 70Hz)
- Control keyboard LEDs (num lock, caps lock, scroll lock)
- Provide utility functions for delays and timer configuration
- Expose lookup tables for ASCII-to-scancode conversion

## Key Types / Data Structures
None (declaration header only).

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| KeyboardQueue | int[256] volatile | global | Circular queue storing raw keyboard events |
| Keyhead | int volatile | global | Write pointer for KeyboardQueue |
| Keytail | int volatile | global | Read pointer for KeyboardQueue |
| Keyboard | int[128] volatile | global | Current pressed state of each scan code |
| Keystate | int[128] volatile | global | Previous key state (likely for change detection) |
| ticcount | int volatile | global | Frame counter, incremented by timer ISR (~70Hz) |
| fasttics | int volatile | global | Possibly faster/secondary timer counter |
| KeyboardStarted | int | global | Flag indicating keyboard ISR is initialized |
| ASCIINames | int[] | global | Lookup: ASCII character → scan code |
| ShiftNames | int[] | global | Lookup: shifted ASCII character → scan code |
| PausePressed | boolean volatile | global | Game pause state flag |
| PanicPressed | boolean volatile | global | Emergency/panic key state flag |

## Key Functions / Methods

### I_StartupTimer
- Signature: `void I_StartupTimer(void)`
- Purpose: Initialize and install the timer ISR
- Inputs: None
- Outputs/Return: None
- Side effects: Hooks hardware timer interrupt, starts ticcount incrementing
- Calls: Not visible in this file
- Notes: Must be called during engine initialization before main loop

### I_SetTimer0
- Signature: `void I_SetTimer0(int speed)`
- Purpose: Reconfigure timer frequency/speed
- Inputs: `speed` – timer divisor or frequency setting
- Outputs/Return: None
- Side effects: Changes ticcount increment rate
- Calls: Not visible in this file
- Notes: Used to adjust frame rate; may be called at runtime

### I_ShutdownTimer
- Signature: `void I_ShutdownTimer(void)`
- Purpose: Uninstall and disable the timer ISR
- Inputs: None
- Outputs/Return: None
- Side effects: Stops ticcount incrementing; restores original timer state
- Calls: Not visible in this file
- Notes: Must be called during engine shutdown

### I_StartupKeyboard
- Signature: `void I_StartupKeyboard(void)`
- Purpose: Initialize and install the keyboard ISR
- Inputs: None
- Outputs/Return: None
- Side effects: Hooks keyboard interrupt, initializes KeyboardQueue, sets KeyboardStarted flag
- Calls: Not visible in this file
- Notes: Required before any keyboard input is processed

### I_ShutdownKeyboard
- Signature: `void I_ShutdownKeyboard(void)`
- Purpose: Uninstall and disable the keyboard ISR
- Inputs: None
- Outputs/Return: None
- Side effects: Stops keyboard interrupt handling; restores original keyboard state
- Calls: Not visible in this file
- Notes: Called during engine shutdown

### I_SetKeyboardLEDs
- Signature: `void I_SetKeyboardLEDs(int which, boolean val)`
- Purpose: Control keyboard indicator lights (num lock, caps lock, scroll lock)
- Inputs: `which` – LED index (0=scroll lock, 1=num lock, 2=caps lock); `val` – on/off
- Outputs/Return: None
- Side effects: Sends command to keyboard controller hardware
- Calls: Not visible in this file
- Notes: Requires keyboard to be initialized

### I_Delay
- Signature: `void I_Delay(int delay)`
- Purpose: Block execution for specified duration
- Inputs: `delay` – time units (ticks or milliseconds, unclear)
- Outputs/Return: None
- Side effects: Busy-waits or sleeps; blocks game thread
- Calls: Not visible in this file
- Notes: Likely polling ticcount in a loop

### I_SendKeyboardData
- Signature: `void I_SendKeyboardData(int val)`
- Purpose: Send raw command/data byte to keyboard controller
- Inputs: `val` – data byte to transmit
- Outputs/Return: None
- Side effects: Writes to keyboard controller port
- Calls: Not visible in this file
- Notes: Low-level hardware communication; typically for LED control or reset

### ISR_SetTime
- Signature: `void ISR_SetTime(int settime)`
- Purpose: Set/reset the ticcount to a specific value
- Inputs: `settime` – new ticcount value
- Outputs/Return: None
- Side effects: Overwrites global ticcount
- Calls: Not visible in this file
- Notes: Used to synchronize or debug timing

## Control Flow Notes
This header is part of the engine's initialization/shutdown sequence and main loop timing. Timer ISR increments `ticcount` at ~70Hz, driving frame pacing. Keyboard ISR populates `KeyboardQueue` on key events and updates `Keyboard[]/Keystate[]` arrays. Game code reads `ticcount` to time frames and polls keyboard state or reads from the queue.

## External Dependencies
- `keyb.h`: Scan code constant definitions (e.g., `sc_Return`, `sc_Escape`)
- Standard C runtime (implied interrupt/low-level hardware access)
