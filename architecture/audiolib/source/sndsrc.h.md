# audiolib/source/sndsrc.h

## File Purpose
Public header for the SoundSource audio device driver module (SNDSRC.C). Declares the API for controlling the SoundSource parallel-port audio device, including initialization, playback control, and memory management for DMA operations.

## Core Responsibilities
- Define error codes returned by SoundSource operations
- Declare public initialization and shutdown functions
- Declare playback control functions (start, stop, rate control)
- Declare parallel port configuration functions (select port, set callback)
- Define port addresses and device constants for multiple hardware configurations
- Declare memory locking/unlocking for DMA-safe buffers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SS_ERRORS` | enum | Error/status codes returned by SoundSource functions |

## Global / File-Static State
None.

## Key Functions / Methods

### SS_Init
- Signature: `int SS_Init( int soundcard )`
- Purpose: Initialize the SoundSource device
- Inputs: `soundcard` — device selection parameter
- Outputs/Return: Status code from `SS_ERRORS` enum
- Side effects: Configures hardware, initializes driver state
- Calls: (not inferable from header)
- Notes: Must be called before playback operations

### SS_Shutdown
- Signature: `void SS_Shutdown( void )`
- Purpose: Cleanly shut down the SoundSource device
- Inputs: None
- Outputs/Return: None
- Side effects: Stops any active playback, releases hardware resources
- Calls: (not inferable from header)
- Notes: Should be called on program exit or device error

### SS_BeginBufferedPlayback
- Signature: `int SS_BeginBufferedPlayback( char *BufferStart, int BufferSize, int NumDivisions, void (*CallBackFunc)(void) )`
- Purpose: Start playback of audio from a ring buffer with periodic callback notification
- Inputs: Buffer pointer, size, number of divisions, callback function pointer
- Outputs/Return: Status code from `SS_ERRORS` enum
- Side effects: Configures DMA, starts playback, callback will fire at division boundaries
- Calls: (not inferable from header)
- Notes: Requires locked memory; callback synchronization point for buffer refill

### SS_StopPlayback
- Signature: `void SS_StopPlayback( void )`
- Purpose: Stop active audio playback
- Inputs: None
- Outputs/Return: None
- Side effects: Halts DMA transfer, silences output
- Calls: (not inferable from header)

### SS_SetPort
- Signature: `int SS_SetPort( int port )`
- Purpose: Select which parallel port the SoundSource is connected to
- Inputs: Port address (0x3bc, 0x378, or 0x278)
- Outputs/Return: Status code from `SS_ERRORS` enum
- Side effects: Reconfigures driver for specified port
- Calls: (not inferable from header)
- Notes: Supports three standard parallel ports plus Tandy configuration

### SS_LockMemory / SS_UnlockMemory
- Signatures: `int SS_LockMemory( void )` / `void SS_UnlockMemory( void )`
- Purpose: Lock audio buffer memory into physical RAM to ensure DMA access doesn't page
- Inputs: None
- Outputs/Return: `SS_LockMemory` returns status code; `SS_UnlockMemory` returns void
- Side effects: Prevents memory paging; must bracket playback operations
- Calls: (not inferable from header)
- Notes: DPMI operations; essential for reliable DMA transfers in protected mode

**Remaining functions** (`SS_GetCurrentPos`, `SS_GetPlaybackRate`, `SS_SetMixMode`, `SS_SetCallBack`, `SS_ErrorString`): Query/configuration helpers—see notes below.

## Control Flow Notes
Typical initialization and playback sequence:
1. `SS_Init()` — initialize hardware
2. `SS_SetPort()` — configure port address if not default
3. `SS_LockMemory()` — lock DMA buffer
4. `SS_BeginBufferedPlayback()` — start playback with callback
5. (Callback fires periodically; application refills ring buffer)
6. `SS_StopPlayback()` — halt playback when done
7. `SS_UnlockMemory()` — release memory lock
8. `SS_Shutdown()` — cleanup on exit

## External Dependencies
- ANSI C standard library (implicit)
- Parallel port hardware; DPMI for protected-mode memory locking (DOS/DPMI environment)
- Direct hardware I/O to parallel ports (addresses 0x3bc, 0x278, 0x378)
- Tandy sound hardware (legacy support)
