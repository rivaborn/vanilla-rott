# rott/fli_def.h

## File Purpose
Defines the binary structures and constants for the FLIC/FLC animation file format (320×200 FLI and variable-resolution FLC). Includes headers for files, frames, and compression chunks, enabling serialization and deserialization of frame-based animations.

## Core Responsibilities
- Define binary layout of FLIC file header with metadata (dimensions, frame count, timing, creation info)
- Define frame and chunk headers for hierarchical file structure
- Enumerate chunk compression/data types (palette, delta, RLE, literal)
- Provide type identifiers and flags for format variant detection

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| FlicHead | struct | Main FLIC file header; contains size, type, frame count, dimensions, timing, flags, and FLC-specific metadata |
| PrefixHead | struct | Optional preprocessing header with subchunk count |
| FrameHead | struct | Individual frame header with chunk count |
| ChunkHead | struct | Compressed data chunk header |
| ChunkTypes | enum | Chunk type identifiers: COLOR_256, DELTA_FLC, COLOR_64, DELTA_FLI, BLACK, BYTE_RUN, LITERAL, PSTAMP |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| FLI_TYPE | #define | global | Type marker for 320×200 FLI files (0xAF11) |
| FLC_TYPE | #define | global | Type marker for variable-resolution FLC files (0xAF12) |
| FLI_FINISHED | #define | global | File completion flag |
| FLI_LOOPED | #define | global | Animation loop flag |
| PREFIX_TYPE | #define | global | Type marker for prefix header (0xF100) |
| FRAME_TYPE | #define | global | Type marker for frame header (0xF1FA) |

## Key Functions / Methods
None. Pure data structure definition.

## Control Flow Notes
Not inferable. This is a specification file consumed by FLIC reader/writer modules during file I/O operations.

## External Dependencies
- Project-defined types: `Long`, `Ushort`, `Short`, `Ulong`, `Char` (defined elsewhere, likely 16/32-bit C types)
- Uses `#pragma pack(1)` for byte-aligned binary struct layout

## Notes
- Copyright notices cite Jim Kent and Dr. Dobb's Journal (March 1993)
- "Reserved" fields indicate format version compatibility placeholders
- FLC extends FLI with creation dates, creator identifiers, and aspect ratio
- Enum values suggest FLI chunk type 12 (DELTA_FLI) vs FLC type 7 (DELTA_FLC) variants
