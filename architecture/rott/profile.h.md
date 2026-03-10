# rott/profile.h

## File Purpose
A minimalist profiling/performance measurement configuration header. Defines preprocessor constants that control profiling behavior throughout the engine.

## Core Responsibilities
- Enable/disable profiling instrumentation via `PROFILE` macro
- Define profiling tick frequency or sampling interval via `PROFILETICS`

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PROFILE` | Macro (int) | Global | Feature flag for profiling; 0 = disabled |
| `PROFILETICS` | Macro (int) | Global | Profiling interval or tick frequency |

## Key Functions / Methods
None.

## Control Flow Notes
Build-time configuration header. Not part of runtime control flow. The `PROFILE` flag (currently 0) likely gates profiling code elsewhere in the codebase via conditional compilation (`#ifdef PROFILE`). Changes to these constants would enable/adjust profiling behavior globally without rewriting profiling instrumentation.

## External Dependencies
- Standard C preprocessor directives only; no external includes
- Licensed under GNU GPL v2

**Notes:**  
- Extremely lightweight—only two constants, both currently inactive/minimal
- `PROFILE=0` suggests profiling is disabled by default (likely omitted from release builds)
- `PROFILETICS=2` purpose not inferable without seeing usage in instrumented code; possibly a tick multiplier, sample rate divisor, or interval threshold
