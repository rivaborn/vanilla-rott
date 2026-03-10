# rott/lumpy.h

## File Purpose
Public header defining typedef structures for graphics and font resources used throughout the ROTT engine. Contains data structure definitions for pictures, fonts, patches (sprites), and bitmap images—all core to the rendering and resource management system.

## Core Responsibilities
- Define in-memory representations of picture/sprite data (pic_t, lpic_t)
- Define font metadata and character rasterization structures (font_t, cfont_t)
- Define patch structures for sprite rendering with column-based offsets (patch_t, transpatch_t)
- Define lossless bitmap image format with embedded palette (lbm_t)
- Provide common type aliases across rendering and resource loading modules

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| pic_t | struct | Simple picture: 8-bit width/height + pixel data |
| lpic_t | struct | Large picture with origin offsets (orgx, orgy) for positioned sprites |
| font_t | struct | Font metrics: height, per-character widths, column offsets, raster data |
| cfont_t | struct | Color font: like font_t but with embedded 768-byte palette (3×256 RGB) |
| lbm_t | struct | LBM image: width, height, 768-byte palette, pixel data |
| patch_t | struct | Sprite patch: bounding box, offsets, column offset table for column-based rendering |
| transpatch_t | struct | Transparent patch: extends patch_t with translevel field for alpha/transparency |

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This is a data definition header—no runtime logic. Types are populated during resource loading (asset deserialization) and consumed by rendering, sprite, and font subsystems during frame updates and draw passes.

## External Dependencies
- Standard C scalar types (byte, short, char, unsigned short)
- No explicit includes visible; assumes standard type definitions available in including translation units

## Notes
- **patch_t / transpatch_t**: The `collumnofs[320]` array uses a fixed size of 320, but documentation notes only `[width]` entries are valid; index `[0]` points to `&collumnofs[width]` (post-header offset table).
- **cfont_t**: Includes inline 768-byte palette; suggests color fonts are self-contained resources.
- **Memory layout**: Structures use packed layouts with flexible array members (`data` at end), enabling single allocation for header + variable-length pixel/raster data.
