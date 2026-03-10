# audiolib/source/interrup.h

## File Purpose
Provides low-level CPU interrupt flag manipulation through inline x86 assembly. Used to disable interrupts before critical sections and restore them afterward, enabling atomic operations in the audio library.

## Core Responsibilities
- Disable CPU interrupts and return current CPU state (EFLAGS)
- Restore CPU interrupts to a previously saved state
- Support interrupt-safe critical sections in the audio system

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### DisableInterrupts
- Signature: `unsigned long DisableInterrupts( void );`
- Purpose: Atomically disable CPU interrupts and return the current interrupt state for later restoration.
- Inputs: None.
- Outputs/Return: Current value of EFLAGS register (containing the interrupt flag and other CPU state).
- Side effects: Clears the CPU interrupt flag (CLI instruction); affects global interrupt behavior.
- Calls: None (inline x86 assembly only).
- Notes: Uses Watcom pragma to embed x86 assembly: `pushfd` (save EFLAGS), `pop eax` (move to return register), `cli` (clear interrupt flag). Returns must be passed to `RestoreInterrupts()` to re-enable interrupts.

### RestoreInterrupts
- Signature: `void RestoreInterrupts( unsigned long flags );`
- Purpose: Atomically restore CPU interrupts and other flags to a previously saved state.
- Inputs: `flags` – EFLAGS value (typically from prior `DisableInterrupts()` call).
- Outputs/Return: None.
- Side effects: Restores the CPU interrupt flag and other EFLAGS bits from the input parameter.
- Calls: None (inline x86 assembly only).
- Notes: Uses Watcom pragma: `push eax` (push flags parameter), `popfd` (restore EFLAGS from stack). Assumes parameter in EAX register via Watcom calling convention.

## Control Flow Notes
These are low-level synchronization primitives called by other audio library modules (e.g., interrupt handlers, timer code) to protect shared state during interrupt-sensitive operations. Typical pattern: `flags = DisableInterrupts(); /* critical section */; RestoreInterrupts(flags);`

## External Dependencies
- **Compiler-specific**: Watcom C `#pragma aux` directives for inline assembly; not portable to other C compilers.
- No external symbols referenced.
