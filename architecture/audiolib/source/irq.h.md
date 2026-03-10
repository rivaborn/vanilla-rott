# audiolib/source/irq.h

## File Purpose
Public header declaring interrupt request (IRQ) management functions for the audio library. Provides the interface for installing and restoring interrupt vector handlers, used during audio system initialization and shutdown to manage hardware interrupts (likely for sound cards).

## Core Responsibilities
- Define error codes for IRQ operations
- Provide validation macro for IRQ numbers
- Declare interrupt vector installation function
- Declare interrupt vector restoration function

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `IRQ_ERRORS` | enum | Status codes for IRQ operations (Warning, Error, Ok) |

## Global / File-Static State
None.

## Key Functions / Methods

### IRQ_SetVector
- Signature: `int IRQ_SetVector(int vector, void (__interrupt *function)(void))`
- Purpose: Install a custom interrupt handler for a specific IRQ vector
- Inputs: 
  - `vector`: IRQ number (0–15)
  - `function`: Pointer to interrupt handler callback (uses `__interrupt` calling convention)
- Outputs/Return: Status code from `IRQ_ERRORS` enum
- Side effects: Modifies interrupt vector table; registers a new interrupt handler
- Calls: Not inferable from header
- Notes: Assumes `vector` is in valid range [0, 15]; validation macro `VALID_IRQ()` provided for callers

### IRQ_RestoreVector
- Signature: `int IRQ_RestoreVector(int vector)`
- Purpose: Restore the original/default interrupt handler for a given IRQ vector
- Inputs: 
  - `vector`: IRQ number to restore
- Outputs/Return: Status code from `IRQ_ERRORS` enum
- Side effects: Modifies interrupt vector table; restores previous handler
- Calls: Not inferable from header
- Notes: Likely called during shutdown to clean up installed handlers

## Control Flow Notes
This is engine initialization/shutdown code. `IRQ_SetVector()` is called during audio subsystem startup to hook the sound card interrupt; `IRQ_RestoreVector()` is called during shutdown to restore the system state. The implementation likely stores the original vector address internally to support restoration.

## External Dependencies
- Uses `__interrupt` calling convention keyword (compiler-specific, likely Watcom C or similar DOS-era compiler)
- Targets low-level x86 interrupt architecture (DOS/real-mode environment based on copyright era and IRQ range 0–15)
