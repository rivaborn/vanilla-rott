# rott/version.h

## File Purpose
Defines compile-time version constants for the ROTT game engine. Provides a single source of truth for version numbering that is #included by the build system and game code. The combined version macro allows version comparisons without string parsing.

## Core Responsibilities
- Define major and minor version numbers as preprocessor constants
- Compute a combined version integer from component versions
- Provide centralized version information for conditional compilation or runtime version reporting
- Enable consistent version numbering across the entire codebase

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ROTTMAJORVERSION` | macro (int) | global | Primary version identifier (currently 1) |
| `ROTTMINORVERSION` | macro (int) | global | Secondary version identifier (currently 4) |
| `ROTTVERSION` | macro (computed int) | global | Combined version as single integer (10×major + minor = 14) |

## Key Functions / Methods
None.

## Control Flow Notes
This is a compile-time header; no control flow involvement. Used by the build system and included by source files that need version information.

## External Dependencies
- GNU GPL v2 license header only; no code dependencies
- Designed to be #included by other translation units
