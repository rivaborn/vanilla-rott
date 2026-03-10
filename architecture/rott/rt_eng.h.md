# rott/rt_eng.h

## File Purpose
Public header declaring the core raycasting function. This appears to be the main rendering engine interface for a raycaster-based game engine, exposing the primary ray-casting routine used for rendering 2D/3D views.

## Core Responsibilities
- Declare the `RayCast` function interface
- Specify x86 register calling conventions via pragma directive for performance-critical rendering code
- Provide public access to the raycasting engine entry point

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### RayCast
- **Signature:**  
  `int RayCast(int count, int xtstep, int ytstep, int offs, int xstep, int ystep)`
  
- **Purpose:**  
  Primary raycasting engine function that casts rays and renders the view. The parameters suggest iterative stepping through a 2D/3D grid.

- **Inputs:**  
  - `count` – likely number of rays or columns to cast
  - `xtstep`, `ytstep` – step increments in x and y directions per iteration
  - `offs` – offset or initial position/buffer offset
  - `xstep`, `ystep` – additional step values (possibly direction or grid size)

- **Outputs/Return:**  
  Returns an `int` (returned in EDI register per pragma)

- **Side effects (global state, I/O, alloc):**  
  Likely modifies frame buffer or screen memory; exact effects depend on implementation.

- **Calls (direct calls visible in this file):**  
  Not visible (implementation in separate file).

- **Notes:**  
  Pragma declares x86 register calling convention (Watcom C syntax): parameters passed via EDI, EAX, EBX, ESI, ECX, EDX; modifies ESI and EDI. This is a hand-optimized assembly function for the hot rendering path.

## Control Flow Notes
This function is invoked during the render phase of each frame. It is the core loop of the raycasting rendering algorithm, likely called once per frame to generate the complete view.

## External Dependencies
- None declared; implementation expected in `rt_eng.c` or similar compiled object file.
