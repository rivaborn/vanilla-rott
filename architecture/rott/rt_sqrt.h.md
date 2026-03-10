# rott/rt_sqrt.h

## File Purpose
Header file declaring two fixed-point square root functions optimized for performance: a low-precision variant (8.8 bit accuracy) and a high-precision variant (8.16 bit accuracy). These are core mathematical utilities for the game engine's physics and graphics calculations.

## Core Responsibilities
- Declare fixed-point square root functions with Watcom C++ inline assembly implementations
- Provide low-precision (`FixedSqrtLP`) for speed-critical paths
- Provide high-precision (`FixedSqrtHP`) for accuracy-critical paths
- Include binary-search-based algorithms optimized for 32-bit x86 architecture

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### FixedSqrtLP
- **Signature:** `long FixedSqrtLP(long n)`
- **Purpose:** Compute square root of a fixed-point number with low precision (8.8 bits).
- **Inputs:** `n` — input fixed-point value (caller passes in `ecx`).
- **Outputs/Return:** Fixed-point square root result (returned in `eax`).
- **Side effects:** None (pure computation).
- **Calls:** None (pure x86 assembly).
- **Notes:** Binary-search algorithm that shifts the result left by 8 bits. Uses registers `eax`, `ebx`, `ecx`, `edx` as working space.

### FixedSqrtHP
- **Signature:** `long FixedSqrtHP(long n)`
- **Purpose:** Compute square root of a fixed-point number with high precision (8.16 bits).
- **Inputs:** `n` — input fixed-point value (caller passes in `ecx`).
- **Outputs/Return:** Fixed-point square root result (returned in `eax`).
- **Side effects:** None (pure computation).
- **Calls:** None (pure x86 assembly).
- **Notes:** Two-stage algorithm: first stage matches `FixedSqrtLP`, second stage (sqrtHP3–sqrtHP6) performs additional iterations with scaled precision. Result shifted left by 16 bits for higher fractional precision.

## Control Flow Notes
These are utility functions called on-demand by game logic and graphics code. Not part of the init/frame/update/render cycle; invoked wherever fixed-point square roots are needed (distance calculations, lighting, etc.).

## External Dependencies
- No includes or imports.
- Uses Watcom C++ `#pragma aux` directive for inline x86 assembly embedding.
- Fixed32 type referenced in comments but not defined in this file (likely defined elsewhere).
