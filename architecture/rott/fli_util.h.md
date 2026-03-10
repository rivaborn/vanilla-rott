# rott/fli_util.h

## File Purpose
Header file defining machine-specific abstractions for the FLI animation/graphics system in the ROTT engine. Provides portable interfaces to screen rendering, keyboard input, timing, file I/O, and large memory allocation (>64K blocks) to isolate platform-dependent code from the FLI decoder.

## Core Responsibilities
- Abstract screen/video hardware operations (open, close, drawing pixels, color palette management)
- Abstract keyboard polling and input handling
- Abstract system clock/timing operations
- Abstract large block memory allocation for DOS platform constraints
- Abstract binary file reading operations
- Aggregate machine initialization/shutdown of all peripheral devices

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Pixel | typedef | Single pixel value (Uchar) |
| Color | struct | RGB color map entry (r, g, b each 0–255) |
| Pixels2 | struct | Two adjacent pixels for word-oriented run-length encoding |
| Screen | struct | Screen device abstraction; holds framebuffer pointer, dimensions, mode state, open flag |
| Clock | struct | Timing abstraction; holds clock speed (ticks/second) |
| Key | struct | Keyboard input; holds ASCII code and scan code |
| MemPtr | typedef | Pointer to large allocated memory block (Uchar *) |
| Machine | struct | Aggregate of Screen, Clock, and Key; represents complete hardware device |

## Global / File-Static State
None.

## Key Functions / Methods

### screen_open
- Signature: `ErrCode screen_open(Screen *s)`
- Purpose: Initialize graphics mode and populate screen structure
- Inputs: Pointer to uninitialized Screen struct
- Outputs/Return: Error code
- Side effects: Sets graphics mode, allocates/maps framebuffer
- Calls: Not visible from header
- Notes: Must be called before any screen operations

### screen_close
- Signature: `void screen_close(Screen *s)`
- Purpose: Restore original display mode and release resources
- Inputs: Pointer to open Screen
- Side effects: Restores video mode, unmaps framebuffer

### screen_put_dot
- Signature: `void screen_put_dot(Screen *s, int x, int y, Pixel color)`
- Purpose: Set a single pixel at given coordinates
- Inputs: Screen, x/y position, color value

### screen_copy_seg
- Signature: `void screen_copy_seg(Screen *s, int x, int y, Pixel *pixels, int count)`
- Purpose: Bulk copy pixel data from memory into screen at position
- Inputs: Screen, x/y start position, pixel array, count of pixels
- Notes: Optimized path for uncompressed pixel runs

### screen_put_colors / screen_put_colors_64
- Signature: `void screen_put_colors(Screen *s, int start, Color *colors, int count)`
- Purpose: Update color palette entries; _64 variant uses 6-bit RGB values
- Inputs: Screen, palette index start, color array, count
- Notes: RGB values 0–255 for standard version, 0–64 for 64-variant

### clock_open / clock_close / clock_ticks
- Signature: `ErrCode clock_open(Clock *c)` | `void clock_close(Clock *c)` | `Ulong clock_ticks(Clock *c)`
- Purpose: Initialize timer, shutdown, and read elapsed ticks
- Inputs: Clock struct pointer
- Outputs/Return: Error code (open), ticks value (read)
- Notes: Ticks are relative to clock->speed

### key_open / key_close / key_ready / key_read
- Signature: `ErrCode key_open(Key *k)` | `void key_close(Key *k)` | `Boolean key_ready(Key *k)` | `Uchar key_read(Key *k)`
- Purpose: Initialize keyboard, shutdown, check if key pending, and read next key
- Outputs/Return: ASCII code (read), boolean (ready)
- Notes: Non-blocking poll pattern

### big_alloc / big_free
- Signature: `ErrCode big_alloc(MemPtr *bb, Ulong size)` | `void big_free(MemPtr *bb)`
- Purpose: Allocate/deallocate large memory blocks (>64K), handling DOS segmentation
- Inputs: Pointer to MemPtr, size in bytes
- Outputs/Return: Error code
- Notes: Abstracts DOS far pointer / segment:offset complexity

### file_open_to_read / file_read_block / file_read_big_block
- Signature: `ErrCode file_open_to_read(FileHandle *phandle, char *name)` | `ErrCode file_read_block(FileHandle handle, void *block, unsigned size)` | `ErrCode file_read_big_block(FileHandle handle, MemPtr bb, Ulong size)`
- Purpose: Open binary file for reading, read fixed and large blocks
- Inputs: File handle pointer/value, buffer, size
- Outputs/Return: Error code; _read_block fails if fewer bytes read than requested
- Notes: _read_big_block supports >64K reads

### machine_open / machine_close
- Signature: `ErrCode machine_open(Machine *m)` | `void machine_close(Machine *m)`
- Purpose: Initialize/shutdown all hardware (screen, clock, keyboard) as unified device
- Inputs: Machine struct pointer
- Outputs/Return: Error code
- Notes: Entry/exit point for hardware lifecycle

## Control Flow Notes
This file defines abstractions only; control flow is determined by implementations. Typical usage:
- **Init**: `machine_open()` opens screen, clock, keyboard
- **Frame loop**: `key_ready()/key_read()` for input, screen drawing ops for render, `clock_ticks()` for timing
- **Shutdown**: `machine_close()`

FLI decoder presumably calls these functions to animate FLI files on hardware.

## External Dependencies
- **Type definitions**: `Uchar`, `Ushort`, `Ulong`, `Boolean`, `ErrCode`, `FileHandle` (defined elsewhere, likely common header)
- **Scope**: Portable abstractions; platform-specific implementations exist elsewhere (DOS, other OS versions)
