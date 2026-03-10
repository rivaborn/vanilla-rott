# rott/_rt_rand.h

## File Purpose

Private header file for the random number generator module. Provides a precomputed lookup table of 2048 pseudo-random byte values used by the game engine's RNG implementation in `rt_rand.c`.

## Core Responsibilities

- Define the size constant for the random table
- Supply a precomputed 2048-byte lookup table of pseudo-random values
- Enable deterministic, fast random number generation without runtime seeding

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `RandomTable` | Global array (unsigned char[2048]) | Precomputed lookup table of pseudo-random byte values |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RandomTable` | unsigned char[2048] | Global | Precomputed deterministic random values indexed by RNG functions |
| `SIZE_OF_RANDOM_TABLE` | #define (2048) | File-scope constant | Defines the fixed size of the random table |

## Key Functions / Methods

None — this file contains only data.

## Control Flow Notes

Not part of active control flow. This is a pure data resource. Typical usage pattern: `rt_rand.c` implements RNG functions that index into `RandomTable` (likely using a modulo or mask operation) to return pseudo-random values. Enables fast, repeatable randomness for gameplay mechanics without expensive entropy computation.

## External Dependencies

- None — self-contained data structure
- Intended for inclusion by `rt_rand.c` only (marked as "private header")
