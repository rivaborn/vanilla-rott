# rott/_isr.h

## File Purpose
Defines interrupt service routine (ISR) constants for x86 interrupt vector assignments. Provides symbolic names for hardware interrupt numbers used by the game engine's interrupt handlers.

## Core Responsibilities
- Define interrupt vector constants for system-level event handling
- Provide ISR-related macro definitions for cross-module consistency
- Support DOS/early Windows interrupt architecture used by the game

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|------|---------|
| TIMERINT | Macro (0x08) | global | x86 interrupt vector for timer (IRQ 0) |
| KEYBOARDINT | Macro (0x09) | global | x86 interrupt vector for keyboard (IRQ 1) |

## Key Functions / Methods
None.

## Control Flow Notes
This is a configuration header consumed at compile-time. The constants define which x86 interrupt vectors the engine hooks for real-time event handling (timer tick updates and keyboard input). Expected usage occurs during ISR setup/initialization phases and in interrupt handler code.

## External Dependencies
- `_isr_private` header guard (file-private; prevents multiple inclusion)
- No external includes or runtime dependencies
