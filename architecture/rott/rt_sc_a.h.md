# rott/rt_sc_a.h

## File Purpose
Header file declaring low-level assembly drawing functions for rendering masked columns (vertical screen strips) during the ROTT rendering pipeline. Provides optimized scanline drawing primitives with explicit register-level calling conventions for x86 assembly implementations.

## Core Responsibilities
- Declare column/scanline drawing functions for masked sprites and walls
- Specify x86 calling conventions and register mappings via `#pragma aux` directives
- Define the interface between C renderer and assembly-optimized drawing code
- Support solid color fills and transparent/translucent column rendering

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### R_DrawColumn
- Signature: `void R_DrawColumn (byte * buf)`
- Purpose: Draw a column (vertical strip) with masked pixels, respecting transparency
- Inputs: `buf` (EDI register) – destination buffer address
- Outputs/Return: None (writes directly to buffer)
- Side effects: Modifies destination framebuffer; modifies registers eax, ebx, ecx, edx, esi, edi
- Calls: (assembly implementation not visible in header)
- Notes: Core masked post/sprite drawing primitive; EDI holds buffer pointer per calling convention

### R_DrawSolidColumn
- Signature: `void R_DrawSolidColumn (int color, byte * buf)`
- Purpose: Draw a solid column filled with a single color
- Inputs: `color` (EBX register), `buf` (EDI register)
- Outputs/Return: None
- Side effects: Writes solid color column to framebuffer; modifies eax, ecx, edi
- Calls: (assembly implementation not visible)
- Notes: Lightweight fill operation; fewer register constraints than masked variant

### R_TransColumn
- Signature: `void R_TransColumn (byte * buf)`
- Purpose: Draw a translucent/transparent column (blended rendering)
- Inputs: `buf` (EDI register) – destination buffer
- Outputs/Return: None
- Side effects: Modifies framebuffer with blending; modifies eax, ebx, esi, edi
- Calls: (assembly implementation not visible)
- Notes: Used for transparency effects; likely uses palette-based or fixed-function blending

### R_DrawClippedColumn
- Signature: `void R_DrawClippedColumn (byte * buf)`
- Purpose: Draw a column with pixel clipping/scissoring support
- Inputs: `buf` (EDI register) – destination buffer
- Outputs/Return: None
- Side effects: Writes clipped column to framebuffer; modifies eax, ebx, ecx, edx, esi, edi
- Calls: (assembly implementation not visible)
- Notes: Register signature matches R_DrawColumn; handles clip rectangle calculations in assembly

## Control Flow Notes
These functions are graphics primitives called during the **render phase** of each frame—likely invoked by the wall/sprite rasterization loop to draw visible vertical segments. The calling convention isolation (via `#pragma aux`) suggests these are performance-critical inner-loop routines optimized for x86 assembly.

## External Dependencies
- Standard C `byte` type (unsigned char)
- Borland C++ `#pragma aux` compiler directive for custom calling conventions
- x86 register model (eax, ebx, ecx, edx, esi, edi)
