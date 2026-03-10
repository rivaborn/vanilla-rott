# rott/fli_main.c

## File Purpose
Implements the FLI/FLC cinematic file player for the ROTT engine. Provides decompression routines for various FLIC frame chunk types and high-level playback control for displaying animated sequences on-screen.

## Core Responsibilities
- Decompress FLIC chunk types (COLOR_256/64, DELTA_FLC/FLI, BYTE_RUN, LITERAL, BLACK)
- Manage FLIC file I/O from disk or in-memory buffer
- Load and parse FLIC headers and frame structures
- Render decompressed frames to screen via pixel/color operations
- Synchronize playback timing with frame delays
- Control cinematic playback (single-play and looping modes)
- Handle user abort (keyboard input during playback)
- Convert between FLI and FLC formats

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Flic` | struct | Main FLIC playback state (header, file handle, offsets, buffer, play mode) |
| `FlicHead` | struct | FLIC file header with metadata, frame count, size, type, timing |
| `FrameHead` | struct | Frame header with chunk count and size |
| `ChunkHead` | struct | Individual chunk header with compression type |
| `Screen` | struct | Display device with pixel buffer and dimensions |
| `Color` | struct | RGB palette entry |
| `Pixels2` | struct | Word-aligned pair of pixels for 16-bit delta encoding |
| `Machine` | struct | Machine state bundling screen, clock, keyboard |
| `ColorOut` | typedef | Function pointer for color palette output callbacks |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `err_strings[]` | `char*[]` | static | Error message lookup table indexed by negative error code |

## Key Functions / Methods

### decode_color
- **Signature:** `static void decode_color(Uchar *data, Flic *flic, Screen *s, ColorOut *output)`
- **Purpose:** Generic color palette decompressor for both COLOR_256 and COLOR_64 chunks. Unpacks compressed palette data and invokes output callback.
- **Inputs:** Compressed chunk data, FLIC state, screen, output function pointer
- **Outputs/Return:** None; writes to screen via callback
- **Side effects:** Updates screen palette via `output` callback
- **Calls:** `screen_put_colors()` or `screen_put_colors_64()` (via function pointer)
- **Notes:** Uses callback pattern to handle 0–63 vs 0–255 color range normalization in single code path

### decode_delta_flc
- **Signature:** `static void decode_delta_flc(Uchar *data, Flic *flic, Screen *s)`
- **Purpose:** FLC-style word-oriented delta decompression. Handles line-skipping and segment encoding with line packing control.
- **Inputs:** Compressed chunk data, FLIC state, screen
- **Outputs/Return:** None
- **Side effects:** Renders pixels/dots to screen
- **Calls:** `screen_copy_seg()`, `screen_repeat_two()`, `screen_put_dot()`
- **Notes:** Uses goto-based line packing state machine; complex format with skip-lines and EOL dot markers; word-aligned operations

### decode_delta_fli
- **Signature:** `static void decode_delta_fli(Uchar *data, Flic *flic, Screen *s)`
- **Purpose:** FLI-style byte-oriented delta decompression. Simpler line-by-line delta encoding with run-length and literal copy ops.
- **Inputs:** Compressed chunk data, FLIC state, screen
- **Outputs/Return:** None
- **Side effects:** Renders pixels to screen
- **Calls:** `screen_copy_seg()`, `screen_repeat_one()`
- **Notes:** Byte-oriented, straightforward per-line encoding; used for 320×200 FLI files

### decode_byte_run
- **Signature:** `static void decode_byte_run(Uchar *data, Flic *flic, Screen *s)`
- **Purpose:** Simple byte-run-length decompression. Processes line-by-line with run/literal opcodes.
- **Inputs:** Compressed chunk data, FLIC state, screen
- **Outputs/Return:** None
- **Side effects:** Renders to screen
- **Calls:** `screen_repeat_one()`, `screen_copy_seg()`
- **Notes:** Standard RLE: positive byte = run count, negative = literal byte count

### decode_frame
- **Signature:** `static ErrCode decode_frame(Flic *flic, FrameHead *frame, Uchar *data, Screen *s)`
- **Purpose:** Main frame decoder dispatch. Iterates chunks and routes to appropriate decompressor.
- **Inputs:** FLIC state, frame header, in-memory frame data, screen
- **Outputs/Return:** `Success` or error code
- **Side effects:** Invokes chunk decoders; renders frame
- **Calls:** All `decode_*` functions based on chunk type
- **Notes:** Dispatch on `ChunkHead.type`; silently ignores unknown chunk types

### flic_open
- **Signature:** `ErrCode flic_open(Flic *flic, char *name, MemPtr buf, Boolean usefile)`
- **Purpose:** Initialize FLIC playback state. Opens file or sets up buffer mode, reads and validates header, seeks to frame 1.
- **Inputs:** Flic struct to initialize, filename, buffer pointer (or NULL), use-file flag
- **Outputs/Return:** Success or error (ErrBadFlic, ErrOpen, ErrRead)
- **Side effects:** Opens file handle, initializes Flic state, calls `flic_close()` on error
- **Calls:** `ClearStruct()`, `SetupFlicAccess()`, `CopyNextFlicBlock()`, `SetFlicOffset()`, `flic_close()`
- **Notes:** Converts FLI speed (70 Hz units) to FLC speed (ms); seeks to `oframe1` on success

### flic_close
- **Signature:** `void flic_close(Flic *flic)`
- **Purpose:** Clean shutdown. Closes file handle and zeros Flic state.
- **Inputs:** Flic struct
- **Outputs/Return:** None
- **Side effects:** Closes file handle (if file mode); zeroes all Flic fields
- **Calls:** `close()` (POSIX), `ClearStruct()`
- **Notes:** Safe to call on already-closed Flic

### flic_next_frame
- **Signature:** `ErrCode flic_next_frame(Flic *flic, Screen *screen)`
- **Purpose:** Load and decode the next frame in sequence. Reads frame header, allocates temp buffer, reads frame data, decodes chunks.
- **Inputs:** FLIC state, screen to render to
- **Outputs/Return:** Success or error
- **Side effects:** Allocates and frees temp buffer, updates screen, advances file/buffer offset
- **Calls:** `CopyNextFlicBlock()`, `big_alloc()`, `big_free()`, `decode_frame()`
- **Notes:** Validates FRAME_TYPE header; frame size includes header

### flic_play_once
- **Signature:** `ErrCode flic_play_once(Flic *flic, Machine *machine)`
- **Purpose:** Play FLIC through once without looping. Iterates all frames with timing sync.
- **Inputs:** FLIC state, machine (for timing/abort check)
- **Outputs/Return:** Success, ErrCancel (user abort), or frame error
- **Side effects:** Modifies screen
- **Calls:** `calc_end_time()`, `flic_next_frame()`, `wait_til()`
- **Notes:** Breaks on first error or user abort; uses frame delay from header

### flic_play_loop
- **Signature:** `ErrCode flic_play_loop(Flic *flic, Machine *machine)`
- **Purpose:** Play FLIC in infinite loop until user abort. Caches frame 2 offset for efficient looping.
- **Inputs:** FLIC state, machine
- **Outputs/Return:** Success (user abort) or frame error
- **Side effects:** Modifies screen; seeks file/buffer offset
- **Calls:** `fill_in_frame2()`, `SetFlicOffset()`, `calc_end_time()`, `wait_til()`, `flic_next_frame()`
- **Notes:** Infinite for loop; frame 1 displayed once, then frames 2–N repeat

### PlayFlic
- **Signature:** `void PlayFlic(char *name, unsigned char *buffer, int usefile, int loop)`
- **Purpose:** Public API entry point. Opens machine/display, opens FLIC, centers on screen, plays once or loops, cleans up.
- **Inputs:** Filename, in-memory buffer pointer, use-file flag (0/1), loop flag (0/1)
- **Outputs/Return:** None (errors logged via `Error()`)
- **Side effects:** Opens/closes display, plays animation, may report errors to user
- **Calls:** `machine_open()`, `flic_open()`, `center_flic()`, `flic_play_once()` or `flic_play_loop()`, `flic_close()`, `machine_close()`, `Error()`
- **Notes:** Top-level public interface; gracefully handles open/close failures

### SetupFlicAccess
- **Signature:** `ErrCode SetupFlicAccess(Flic *flic)`
- **Purpose:** Initialize file or buffer access mode based on usefile flag.
- **Inputs:** Flic struct with `usefile` and `name` fields set
- **Outputs/Return:** Success or file error
- **Side effects:** Opens file handle or resets buffer offset
- **Calls:** `file_open_to_read()`
- **Notes:** Abstraction for dual file/buffer mode

### CopyNextFlicBlock
- **Signature:** `ErrCode CopyNextFlicBlock(Flic *flic, MemPtr buf, Ulong size)`
- **Purpose:** Read next block of FLIC data from file or buffer.
- **Inputs:** Flic state, destination buffer, size
- **Outputs/Return:** Success or read error
- **Side effects:** Updates file handle position or buffer offset
- **Calls:** `file_read_big_block()` or `memcpy()`
- **Notes:** Dual-mode I/O; handles >64K blocks

### SetFlicOffset
- **Signature:** `void SetFlicOffset(Flic *flic, Ulong offset)`
- **Purpose:** Seek to absolute offset in FLIC data (file or buffer).
- **Inputs:** Flic state, offset
- **Outputs/Return:** None
- **Side effects:** Updates file position (lseek) or buffer offset
- **Calls:** `lseek()`
- **Notes:** No-op on errors; buffer mode updates offset silently

### flic_err_string
- **Signature:** `char *flic_err_string(ErrCode err)`
- **Purpose:** Convert error code to human-readable string. Defers to OS errno for I/O errors.
- **Inputs:** Error code
- **Outputs/Return:** String pointer (static or from strerror)
- **Side effects:** None
- **Calls:** `strerror()` (system), array lookup
- **Notes:** Inverts negative codes for array indexing; handles unknown codes gracefully

### center_flic
- **Signature:** `static void center_flic(Flic *flic, Screen *s)`
- **Purpose:** Calculate x/y offsets to center FLIC on screen.
- **Inputs:** Flic state, screen
- **Outputs/Return:** None
- **Side effects:** Writes `xoff`, `yoff` fields in Flic
- **Calls:** `screen_width()`, `screen_height()`
- **Notes:** Simple arithmetic; casts to signed to handle unsigned dimensions safely

### fill_in_frame2
- **Signature:** `static ErrCode fill_in_frame2(Flic *flic)`
- **Purpose:** Locate frame 2 offset by reading frame 1 header. Enables efficient loop restart.
- **Inputs:** Flic state (with `oframe1` set)
- **Outputs/Return:** Success or read error
- **Side effects:** Writes `oframe2` field in Flic; seeks in file/buffer
- **Calls:** `SetFlicOffset()`, `CopyNextFlicBlock()`
- **Notes:** Called lazily from `flic_play_loop()` if `oframe2` is 0

### calc_end_time
- **Signature:** `static Ulong calc_end_time(Ulong millis)`
- **Purpose:** Calculate absolute wake-up time for frame timing.
- **Inputs:** Frame delay in milliseconds
- **Outputs/Return:** Absolute time (cinematic clock units)
- **Side effects:** None (reads `GetCinematicTime()`)
- **Calls:** `GetCinematicTime()`
- **Notes:** Scales milliseconds by CLOCKSPEED (VBLCOUNTER)

### wait_til
- **Signature:** `static ErrCode wait_til(Ulong end_time, Machine *machine)`
- **Purpose:** Busy-wait until absolute time or user abort, polling keyboard.
- **Inputs:** Target time, machine (for abort check)
- **Outputs/Return:** Success (timed out) or ErrCancel (abort)
- **Side effects:** Polls keyboard via `CinematicAbort()`
- **Calls:** `CinematicAbort()`, `GetCinematicTime()`
- **Notes:** Machine parameter unused; pure polling loop

### decode_black
- **Signature:** `static void decode_black(Uchar *data, Flic *flic, Screen *s)`
- **Purpose:** Fill entire frame with color 0 (black).
- **Inputs:** Unused data, FLIC state, screen
- **Outputs/Return:** None
- **Side effects:** Fills screen with black
- **Calls:** `screen_repeat_two()`, `screen_put_dot()`
- **Notes:** Handles odd-width frames by setting last pixel separately

### decode_literal
- **Signature:** `static void decode_literal(Uchar *data, Flic *flic, Screen *s)`
- **Purpose:** Copy uncompressed frame data line-by-line to screen.
- **Inputs:** Raw pixel data, FLIC state, screen
- **Outputs/Return:** None
- **Side effects:** Renders frame
- **Calls:** `screen_copy_seg()`
- **Notes:** Simple line-by-line memcpy; no decompression

## Control Flow Notes
**Initialization → Playback → Shutdown:**

1. `PlayFlic()` opens machine (display, clock, keyboard) and FLIC file/buffer
2. `flic_open()` validates header, handles FLI↔FLC conversion, seeks to frame 1
3. `flic_play_once()` or `flic_play_loop()` iterates frames:
   - `flic_next_frame()` → loads frame data → `decode_frame()` → chunk-type decoders
   - `calc_end_time()` schedules next frame wake-up
   - `wait_til()` polls for abort or timeout
4. On abort or error, playback returns control to `PlayFlic()`
5. `flic_close()` and `machine_close()` tear down state

Fits into engine's cinematic/animation subsystem; operates at playback level above low-level screen device drivers.

## External Dependencies
- **Game engine internals:** `cin_glob.h` (cinematic globals, timing), `rt_def.h`, `rt_util.h`, `isr.h`
- **Standard C:** `errno.h` (I/O errors), `string.h` (memcpy), `io.h` (file I/O)
- **FLIC format definitions:** `fli_type.h` (basic C types), `fli_util.h` (Screen, Machine, I/O, Color), `fli_def.h` (FLIC headers/chunks), `fli_main.h` (Flic struct, error codes, macros)
- **Memory management:** `memcheck.h` (likely debug/leak tracking)
- **External symbols (defined elsewhere):**
  - Screen: `screen_open()`, `screen_close()`, `screen_put_dot()`, `screen_copy_seg()`, `screen_repeat_one()`, `screen_repeat_two()`, `screen_put_colors()`, `screen_put_colors_64()`, `screen_width()`, `screen_height()`
  - Machine/Clock/Key: `machine_open()`, `machine_close()`
  - Cinematic: `GetCinematicTime()`, `CinematicAbort()`, `GetCinematicTime()`
  - I/O: `file_open_to_read()`, `file_read_big_block()`, `big_alloc()`, `big_free()`, `lseek()`, `close()`
  - Error reporting: `Error()` (printf-style logging)
