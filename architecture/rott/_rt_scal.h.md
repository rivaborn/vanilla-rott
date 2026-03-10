# rott/_rt_scal.h

## File Purpose
Header file defining scaling and sizing constants for the rendering system. Specifically provides the player height constant used in viewport and geometry calculations. Part of the internal rendering architecture (indicated by the "_private" guard naming).

## Core Responsibilities
- Define player height constant for rendering calculations
- Establish the scaling basis for player-relative rendering
- Provide a single point of definition for geometry constants

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Macros

### PLAYERHEIGHT
- Definition: `(260<<HEIGHTFRACTION)`
- Purpose: Defines the standard player height in scaled units
- Usage: Used as a reference value for player viewport height and related rendering geometry calculations
- Notes: The bit-shift operation `<<HEIGHTFRACTION` suggests HEIGHTFRACTION is a constant defining a fixed-point scale factor; the raw value is 260 units, shifted by the fraction bits for precision

## Control Flow Notes
This header is included during compilation by rendering modules that need to reference player height. The macro is typically used in viewport setup, geometry projection, and camera calculations during frame rendering.

## External Dependencies
- `HEIGHTFRACTION` – defined elsewhere (likely in a related scaling header or math constants file)
