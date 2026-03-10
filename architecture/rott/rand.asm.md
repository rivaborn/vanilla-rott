# rott/rand.asm

## File Purpose
Implements table-based random number generation via lookup table and index cycling. Provides initialization with optional time-based seeding and fast O(1) random value retrieval (0–255) suitable for real-time gameplay.

## Core Responsibilities
- Maintain a 256-entry precomputed random value lookup table
- Initialize the RNG with deterministic (index=0) or time-seeded (via DOS INT 21h) start
- Provide fast random number generation by table lookup and index cycling
- Track and wrap the current table index (0–255)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| _rndtable | DWORD array (256 entries) | Precomputed random values for fast lookup |
| _rndindex | DWORD variable | Current position in table (0–255) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| _rndindex | DWORD | Global | Current table index; wraps via AND 0FFh |
| _rndtable | DWORD[256] | Public global | Lookup table exported for external use |
| _LastRnd | DWORD | File-static | Declared but unused; vestigial |

## Key Functions / Methods

### US_InitRndT_
- **Signature:** `void US_InitRndT_(int randomize)` (randomize in EAX)
- **Purpose:** Seed the random generator; either deterministic or via system time
- **Inputs:** EAX = 0 (deterministic) or non-zero (seed from system time)
- **Outputs/Return:** None
- **Side effects:** Sets _rndindex; calls INT 21h (GetSystemTime) if randomize ≠ 0
- **Calls:** INT 21h (DOS interrupt for system time)
- **Notes:** Deterministic mode (EAX=0) always starts at index 0 for reproducible sequences; randomized mode masks system time (CL seconds register) to 8-bit range

### US_RndT_
- **Signature:** `int US_RndT_(void)` → EAX
- **Purpose:** Return next random value (0–255) and advance table index
- **Inputs:** None
- **Outputs/Return:** EAX = random value (0–255)
- **Side effects:** Increments _rndindex with wraparound (AND 0FFh)
- **Calls:** None
- **Notes:** Trivial linear cycling through precomputed table; extremely fast (~4 cycles). Masks to 8-bit even though DWORD reads are performed (likely for cache locality or legacy reasons).

## Control Flow Notes
Called during engine **initialization** phase (`US_InitRndT_` at startup/level load) and **frame updates** whenever random values are needed (procedural generation, AI, particle effects). No shutdown sequence.

## External Dependencies
- INT 21h (DOS/BIOS system time interrupt) via `US_InitRndT_`
- x86 32-bit processor mode (`.386p` directive)
- Flat memory model (`.model flat`)
- No C runtime or external symbol dependencies
