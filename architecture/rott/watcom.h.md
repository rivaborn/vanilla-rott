# rott/watcom.h

## File Purpose
Provides optimized fixed-point arithmetic operations via Watcom C compiler inline assembly pragmas. This utility header enables efficient integer-based fixed-point math (avoiding floating-point overhead) for core game calculations like transformation, scaling, and division on DOS/early hardware platforms.

## Core Responsibilities
- Declare four fixed-point arithmetic functions (multiply, divide, scale)
- Supply Watcom-specific inline assembly implementations using `#pragma aux`
- Abstract hardware-specific fixed-point operations behind portable C function signatures
- Provide rounding and scaling compensation in low-level arithmetic kernels

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `fixed` | typedef | Fixed-point numeric type (likely 16.16 format: 16-bit integer + 16-bit fractional); declared elsewhere |

## Global / File-Static State
None.

## Key Functions / Methods

### FixedMul
- **Signature:** `fixed FixedMul(fixed a, fixed b);`
- **Purpose:** Multiply two fixed-point numbers with proper scaling and rounding.
- **Inputs:** `a` (fixed operand in `[eax]`), `b` (fixed operand in `[ebx]`)
- **Outputs/Return:** Fixed-point product in `[eax]`
- **Side effects:** Inline assembly modifies `eax`, `edx` registers.
- **Calls:** None (inline assembly only).
- **Notes:** Assembly: `imul ebx` produces 64-bit result in `edx:eax`, add `0x8000` for rounding, shift right by 16 bits to rescale. Assumes 16.16 fixed-point format.

### FixedDiv2
- **Signature:** `fixed FixedDiv2(fixed a, fixed b);`
- **Purpose:** Divide one fixed-point number by another with proper scaling.
- **Inputs:** `a` (fixed dividend in `[eax]`), `b` (fixed divisor in `[ebx]`)
- **Outputs/Return:** Fixed-point quotient in `[eax]`
- **Side effects:** Inline assembly modifies `eax`, `edx` registers.
- **Calls:** None (inline assembly only).
- **Notes:** Assembly: sign-extend `eax` to `edx:eax` (`cdq`), shift left by 16 bits to scale dividend, then signed divide (`idiv ebx`). Assumes 16.16 format.

### FixedMulShift
- **Signature:** `fixed FixedMulShift(fixed a, fixed b, fixed shift);`
- **Purpose:** Multiply two fixed-point numbers and shift result by variable amount.
- **Inputs:** `a` (fixed in `[eax]`), `b` (fixed in `[ebx]`), `shift` (bit count in `[ecx]`)
- **Outputs/Return:** Shifted product in `[eax]`
- **Side effects:** Inline assembly modifies `eax`, `edx` registers.
- **Calls:** None (inline assembly only).
- **Notes:** Assembly: `imul ebx` into `edx:eax`, then `shrd eax,edx,cl` shifts by variable amount. Enables dynamic scaling.

### FixedScale
- **Signature:** `fixed FixedScale(fixed orig, fixed factor, fixed divisor);`
- **Purpose:** Scale a fixed-point number by the ratio `factor / divisor`.
- **Inputs:** `orig` (fixed value in `[eax]`), `factor` (fixed multiplier in `[ebx]`), `divisor` (fixed denominator in `[ecx]`)
- **Outputs/Return:** Scaled result in `[eax]`
- **Side effects:** Inline assembly modifies `eax`, `edx` registers.
- **Calls:** None (inline assembly only).
- **Notes:** Assembly: `imul ebx` then `idiv ecx`—fuses multiply and divide into one inline sequence. Avoids intermediate overflow.

## Control Flow Notes
Not applicable; this is a pure utility header. Functions are called by rendering and game logic modules that perform geometric and physics calculations.

## External Dependencies
- **Type `fixed`** is defined elsewhere (not in this file).
- **Only active when `__WATCOMC__` is defined;** function bodies are inline assembly pragmas specific to Watcom C. Other compilers would require alternate implementations.
