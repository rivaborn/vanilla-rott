# audiolib/source/gus.c

## File Purpose
Provides initialization and shutdown routines for the Gravis Ultrasound (GUS) sound card hardware. Manages GUS OS loading, conventional memory allocation for DMA buffers, and error reporting for both MIDI and digital audio playback.

## Core Responsibilities
- Initialize and shut down GUS hardware with reference counting
- Allocate DOS conventional memory for DMA operations via DPMI
- Map error codes to human-readable error messages
- Integrate GUS-specific initialization into broader audio library (gusmidi, guswave)
- Query and cache available GUS DRAM configuration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `gf1_dma_buff` | struct | DMA buffer descriptor with virtual and physical addresses |
| `load_os` | struct | GUS hardware configuration (IRQs, DMA channels, voice count) |
| `VoiceNode` | struct | Voice state (referenced externally; defined in guswave) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `GUS_HoldBuffer` | `struct gf1_dma_buff` | global | DMA buffer for patch loading |
| `HoldBufferAllocated` | `static int` | static | Flag tracking whether DMA buffer has been allocated |
| `GUS_Installed` | `static int` | static | Reference counter for nested init/shutdown calls |
| `GUS_TotalMemory` | `unsigned long` | global | Total bytes of DRAM available on GUS card |
| `GUS_MemConfig` | `int` | global | Memory configuration bits derived from available DRAM |
| `GUS_ErrorCode` | `int` | global | Last error code encountered |
| `GUS_AuxError` | `int` | global | Auxiliary error code (used by GF1 layer) |

## Key Functions / Methods

### GUS_ErrorString
- **Signature:** `char *GUS_ErrorString(int ErrorNumber)`
- **Purpose:** Maps GUS error codes to human-readable error description strings.
- **Inputs:** `ErrorNumber` — error code (e.g., GUS_Ok, GUS_OutOfMemory, GUS_GF1Error)
- **Outputs/Return:** Pointer to static error message string
- **Side effects:** Recursively calls itself for generic error codes (GUS_Warning, GUS_Error); accesses `GUS_ErrorCode` and `GUS_AuxError` globals
- **Calls:** `gf1_error_str()`, `strerror()`
- **Notes:** Uses switch dispatch; handles file errors via `strerror(GUS_AuxError)`; GUS_Warning and GUS_Error redirect to current `GUS_ErrorCode`

### D32DosMemAlloc
- **Signature:** `void *D32DosMemAlloc(unsigned size)`
- **Purpose:** Allocates a block of conventional (DOS) memory suitable for DMA operations via DPMI.
- **Inputs:** `size` — number of bytes to allocate
- **Outputs/Return:** Virtual pointer to allocated memory; `NULL` if allocation fails
- **Side effects:** Invokes DPMI interrupt 0x31 (allocate DOS memory); sets/clears carry flag in CPU flags
- **Calls:** `int386()` (DPMI interrupt)
- **Notes:** Converts size to paragraphs (16-byte units) via right-shift; returns physical address in EAX register; used for DMA-compatible buffers

### GUS_Init
- **Signature:** `int GUS_Init(void)`
- **Purpose:** Initializes the Gravis Ultrasound hardware for sound and MIDI playback; allocates DMA buffer and loads GUS OS.
- **Inputs:** None
- **Outputs/Return:** `GUS_Ok` on success; `GUS_Error` with error code set on failure
- **Side effects:** Allocates DMA buffer (once); loads GUS OS; populates `GUS_TotalMemory`, `GUS_MemConfig`, `GUS_Installed` globals
- **Calls:** `GetUltraCfg()`, `D32DosMemAlloc()`, `gf1_load_os()`, `gf1_mem_avail()`, `GUS_SetErrorCode()` macro
- **Notes:** Reference-counted initialization (multiple calls increment `GUS_Installed` counter); validates IRQ ≤ 7; DMA buffer allocated only on first init; sets 24 voices in `load_os`

### GUS_Shutdown
- **Signature:** `void GUS_Shutdown(void)`
- **Purpose:** Decrements GUS initialization reference counter; unloads GUS OS when counter reaches zero.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Decrements `GUS_Installed`; calls `gf1_unload_os()` when count reaches 0
- **Calls:** `gf1_unload_os()`
- **Notes:** Must be called same number of times as `GUS_Init()` to fully shut down hardware; designed for nested init/shutdown pairs

### GUSWAVE_Shutdown
- **Signature:** `void GUSWAVE_Shutdown(void)`
- **Purpose:** Terminates digital sound (wave) playback on GUS; stops all active voices and frees allocated voice memory.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `GUSWAVE_Installed` flag; frees all voice memory blocks; calls `GUS_Shutdown()`
- **Calls:** `GUSWAVE_KillAllVoices()`, `gf1_free()`, `GUS_Shutdown()`
- **Notes:** Iterates `VOICES` array; null-checks before freeing; guards execution with `GUSWAVE_Installed` flag

## Control Flow Notes
**Initialization sequence (frame 0):**
- `GUS_Init()` called to initialize hardware; allocates DMA buffer and loads GUS OS
- `GUSWAVE_Shutdown()` called during shutdown; stops all voices and releases resources
- Reference counting allows multiple subsystems to independently init/shutdown

**Runtime:**
- GUS hardware state persists for duration of application
- DMA buffer remains allocated until process exit (not freed in GUS_Shutdown)

## External Dependencies
- **Interrupt/Memory:** `int386()`, `union REGS` from `dos.h` — DPMI calls for DOS memory allocation
- **GF1 Hardware Layer:** `gf1_load_os()`, `gf1_unload_os()`, `gf1_mem_avail()`, `gf1_error_str()`, `gf1_free()` from `newgf1.h`
- **Configuration:** `GetUltraCfg()` — loads ULTRAMID.INI; location defined elsewhere
- **Wave Playback:** `GUSWAVE_Voices[]`, `GUSWAVE_Installed`, `GUSWAVE_KillAllVoices()` from `guswave.h` / `_guswave.h`
- **Standard Library:** `strerror()` from `string.h`
