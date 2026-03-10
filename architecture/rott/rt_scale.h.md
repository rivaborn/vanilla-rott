# rott/rt_scale.h

## File Purpose
Header declaring sprite scaling and 2D projection functions for the Rise of the Triad 3D renderer. Handles conversion of 3D sprite objects to screen-space, with support for scaling, transparency, and lighting effects. Critical bridge between visibility culling (rt_draw.h) and the column-based software rasterizer.

## Core Responsibilities
- Declare scaling entry points for 3D sprites to 2D screen projection
- Export vertical column (post) rendering functions for scaled sprites
- Manage transparency and masking parameters during rasterization
- Provide weapon and HUD sprite drawing functions
- Handle lighting level calculation based on sprite depth/height
- Expose global scaling state (inverse scale, texture coordinates, clipping bounds)

## Key Types / Data Structures
None (uses `visobj_t` from rt_draw.h).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| dc_texturemid | int | global | Vertical texture coordinate midpoint for current column |
| dc_iscale | int | global | Inverse scale factor (1/scale) for texture stepping |
| dc_invscale | int | global | Inverted scale value for coordinate mapping |
| centeryclipped | int | global | Center Y-axis clipped to viewport bounds |
| sprtopoffset | int | global | Y-offset of sprite top from screen center |
| dc_yl, dc_yh | int | global | Lower and upper Y clipping boundaries for column drawing |
| dc_source | byte* | global | Pointer to current texture/sprite data buffer |
| transparentlevel | int | global | Alpha/transparency level (0–255 or similar) for blended sprites |

## Key Functions / Methods

### ScaleShape
- Signature: `void ScaleShape(visobj_t * vis)`
- Purpose: Main entry point to scale and draw a 3D sprite object to screen space
- Inputs: Pointer to visible object (visobj_t) containing projection data
- Outputs/Return: None (renders directly to framebuffer)
- Side effects: Modifies global scaling state (dc_* variables); writes to framebuffer
- Calls: Likely delegates to ScaleSolidShape or ScaleTransparentShape based on sprite flags
- Notes: Central dispatcher for sprite rendering; path likely chosen by transparency or masking flags

### ScaleWeapon
- Signature: `void ScaleWeapon(int xcent, int yoffset, int shapenum)`
- Purpose: Draw a weapon sprite at a fixed screen position with scaling
- Inputs: xcent (X center), yoffset (Y offset from center), shapenum (sprite lump ID)
- Outputs/Return: None
- Side effects: Writes to framebuffer; updates dc_* globals
- Calls: Not inferable
- Notes: HUD/weapon rendering, likely bypasses 3D projection math

### DrawScreenSprite
- Signature: `void DrawScreenSprite(int x, int y, int shapenum)`
- Purpose: Draw a sprite at a fixed screen position without scaling
- Inputs: x, y (screen coordinates), shapenum (sprite resource ID)
- Outputs/Return: None
- Side effects: Renders to framebuffer
- Calls: Not inferable
- Notes: Direct screen-space drawing; used for HUD, UI, or menu sprites

### SetLightLevel
- Signature: `void SetLightLevel(int height)`
- Purpose: Adjust shading/lighting based on sprite depth
- Inputs: height (sprite vertical position or depth)
- Outputs/Return: None
- Side effects: Modifies shading or colormap applied to subsequent draws
- Calls: Not inferable
- Notes: Perspective-based lighting; called before drawing scaled sprites

### ScaleMaskedPost
- Signature: `void ScaleMaskedPost(byte * src, byte * buf)`
- Purpose: Draw a single scaled column with masked (transparent) pixels
- Inputs: src (source sprite column data), buf (destination framebuffer)
- Outputs/Return: None
- Side effects: Writes to framebuffer; uses dc_* globals for scaling parameters
- Calls: Not inferable
- Notes: Inner loop for masked sprite rendering; dc_source, dc_iscale, dc_yl/dc_yh must be preset

### ScaleTransparentPost
- Signature: `void ScaleTransparentPost(byte * src, byte * buf, int level)`
- Purpose: Draw a scaled column with blended transparency
- Inputs: src (sprite column), buf (framebuffer), level (transparency intensity)
- Outputs/Return: None
- Side effects: Blends sprite pixels into framebuffer
- Calls: Not inferable
- Notes: Used for ghosts, effects, or partial transparency; level parameter overrides global transparentlevel

### ScaleTransparentShape, ScaleSolidShape
- Purpose: Wrapper functions routing scaled sprite drawing to either transparent or opaque rendering paths
- Notes: Likely loop over columns calling ScaleMaskedPost or ScaleTransparentPost

## Control Flow Notes
Part of the frame rendering pipeline after 3D visibility/raycasting (ThreeDRefresh in rt_draw.h). Executed during the sprite-to-screen projection phase: visibility list is built → sprites sorted by depth → ScaleShape called per visible sprite → columns drawn via ScaleMaskedPost/ScaleTransparentPost → framebuffer flipped.

## External Dependencies
- **rt_draw.h**: visobj_t structure, global projection state (viewx, viewy, viewangle, lights), frame variables (tics, levelheight), math tables (sintable, costable, tantable)
- Implied: Texture/sprite resource manager (lump system), framebuffer manager, shading tables
