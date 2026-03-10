# rott/rt_error.h

## File Purpose
Public interface for the error management system in the Rise of the Triad engine. Provides initialization/shutdown routines and exposes a global flag for tracking division-by-zero errors at runtime.

## Core Responsibilities
- Declare global error state variables (`DivisionError`)
- Provide startup/shutdown entry points for error subsystem initialization
- Define the public API for engine error handling

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DivisionError` | `boolean` | global | Flag to track division-by-zero error conditions |

## Key Functions / Methods

### UL_ErrorStartup
- Signature: `void UL_ErrorStartup(void)`
- Purpose: Initialize the error handling subsystem
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up error tracking state; likely initializes handlers or resets error flags
- Calls: Not inferable from this file
- Notes: Paired with `UL_ErrorShutdown`; called during engine initialization

### UL_ErrorShutdown
- Signature: `void UL_ErrorShutdown(void)`
- Purpose: Shut down the error handling subsystem
- Inputs: None
- Outputs/Return: None
- Side effects: Cleans up error subsystem state
- Calls: Not inferable from this file
- Notes: Paired with `UL_ErrorStartup`; called during engine shutdown

## Control Flow Notes
This module fits into the engine's **init/shutdown** phase. `UL_ErrorStartup()` is called early during engine startup, and `UL_ErrorShutdown()` during cleanup. The `DivisionError` flag is likely checked during arithmetic operations or frame updates to detect runtime errors.

## External Dependencies
- `boolean` type (defined elsewhere in engine, likely a typedef)
- No includes visible in this header; implementation file presumably includes necessary system/engine headers
