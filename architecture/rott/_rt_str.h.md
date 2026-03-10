# rott/_rt_str.h

## File Purpose
Private header for string drawing and measurement utilities in the rendering system. Declares function prototypes for proportional string operations and initializes global function pointers used throughout the engine for text rendering.

## Core Responsibilities
- Declare prototypes for proportional string drawing (`VWB_DrawPropString`)
- Declare prototypes for proportional string measurement (`VW_MeasurePropString`)
- Define and initialize global function pointers for abstracted string operations (`USL_MeasureString`, `USL_DrawString`)
- Provide a layer of indirection for string rendering across the engine

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `font_t` | typedef | Font definition (defined elsewhere) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `USL_MeasureString` | `void (*)(char *, int *, int *, font_t *)` | Global | Function pointer for measuring string dimensions (width/height) |
| `USL_DrawString` | `void (*)(char *)` | Global | Function pointer for rendering a string to video buffer |

## Key Functions / Methods

### VWB_DrawPropString
- Signature: `void VWB_DrawPropString(char *string)`
- Purpose: Draw a proportional string to the video buffer
- Inputs: `string` – null-terminated C string
- Outputs/Return: None (void)
- Side effects: Writes to video buffer
- Calls: Not inferable from this file
- Notes: Implementation defined in RT_STR.C; assigned to `USL_DrawString` function pointer

### VW_MeasurePropString
- Signature: `void VW_MeasurePropString(char *string, int *width, int *height)`
- Purpose: Calculate width and height dimensions of a proportional string
- Inputs: `string` – text to measure; `width`, `height` – output pointers
- Outputs/Return: Writes calculated dimensions to `*width` and `*height`
- Side effects: None (pure calculation)
- Calls: Not inferable from this file
- Notes: Implementation defined in RT_STR.C; cast and assigned to `USL_MeasureString` function pointer

## Control Flow Notes
Part of the UI/rendering subsystem. The function pointers enable engine code to call string operations through a common interface (`USL_*`) without direct coupling to the proportional string implementation. This allows for runtime selection of string rendering strategies.

## External Dependencies
- `font_t` type (defined elsewhere)
- Actual implementations of `VWB_DrawPropString` and `VW_MeasurePropString` (RT_STR.C)
- Video buffer interface (implicit in `VWB_*` naming convention)
