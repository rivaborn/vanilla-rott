# rott/_rt_msg.h

## File Purpose
Private header file that defines timing constants for the message display system. Provides a single timing parameter used across the engine for message duration management.

## Core Responsibilities
- Define message display timing constant

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MESSAGETIME | int (macro constant) | global | Message display duration in ticks (35 × 6 = 210 ticks) |

## Key Functions / Methods
None.

## Control Flow Notes
This header is purely declarative—used via `#include` wherever message timing is needed. No control flow inferable.

## External Dependencies
None—self-contained constant definition with only copyright headers.
