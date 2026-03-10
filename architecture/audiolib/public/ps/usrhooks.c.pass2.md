# audiolib/public/ps/usrhooks.c — Enhanced Analysis

## Architectural Role

This module is the **mandatory memory allocation checkpoint** for the entire audio library. Every dynamic allocation across all audio subsystems (BLASTER, FM/OPL synthesis, GUS, MIDI, etc.) is routed through these hook functions. By design, it's left modifiable to allow the host application (the game engine) to control audio memory allocation—critical in constrained DOS/early-Windows environments where memory pools and alignment requirements vary. This follows the classic "hooking" pattern for providing customization points without coupling the library to specific allocators.

## Key Cross-References

### Incoming (who depends on this file)
While the cross-reference index provided doesn't enumerate the callers (likely due to the large breadth of the audiolib subsystem), the architectural pattern makes clear that **all memory allocation across audiolib calls these functions**:
- All device driver modules (`blaster.c`, `gus.c`, `awe32.c`, etc.)
- All synthesis engines (`adlibfx.c`, `al_midi.c`)
- Any module handling buffers or voice data structures

The modularity suggests callers should not directly call `malloc`/`free`; instead, they call `USRHOOKS_GetMem`/`USRHOOKS_FreeMem`.

### Outgoing (what this file depends on)
- `<stdlib.h>` (`malloc`, `free`) — direct system allocators
- `usrhooks.h` — interface definition and error code enum (`USRHOOKS_Ok`, `USRHOOKS_Error`)

## Design Patterns & Rationale

1. **Hook/Customization Pattern**: The module is explicitly marked "left public for you to modify." This allows host applications to:
   - Substitute custom allocators (e.g., from a game's memory pool)
   - Add instrumentation (tracking, leak detection)
   - Enforce alignment or size constraints
   
2. **Error Code Return Pattern**: Returns `int` status codes instead of throwing exceptions (typical for C and pre-C++ audio libraries). The output is passed via `void **ptr` rather than returned directly, freeing the return value for error signaling.

3. **Dword-Alignment Guarantee**: The comment notes allocated memory is "assumed to be dword aligned"—critical for:
   - DMA transfers to sound hardware (many sound cards required alignment)
   - Direct hardware writes from audio buffers
   - MIDI/FM synthesis voice structures that may be accessed by interrupt handlers

## Data Flow Through This File

**Allocation flow:**
- Caller invokes `USRHOOKS_GetMem(void **ptr, unsigned long size)`
- Size requested → passed to `malloc()`
- On success: `*ptr` populated with heap address, return `USRHOOKS_Ok`
- On failure (NULL): return `USRHOOKS_Error`, `*ptr` untouched
- Caller checks return code and proceeds or bails

**Deallocation flow:**
- Caller invokes `USRHOOKS_FreeMem(void *ptr)`
- NULL check performed (returns error if NULL)
- Non-NULL: `free(ptr)` called immediately
- Return `USRHOOKS_Ok` on success

No state is maintained; functions are stateless utilities.

## Learning Notes

**1990s Audiolib Design Philosophy:**
- **No memory tracking**: Unlike modern engines, there's no registry of allocated blocks, refcounting, or lifetime management. The caller is fully responsible.
- **Minimal abstraction**: This is barely more than a thin wrapper; the game engine could patch `usrhooks.c` entirely if needed.
- **Hardware-aware alignment**: The explicit dword-alignment note reflects the era's tight coupling to ISA sound cards (Gravis Ultrasound, Blaster Pro, AWE32) that had strict DMA requirements.
- **Extension via substitution**: Rather than allocator strategies (modern approach), the design expects users to replace the entire `.c` file with a custom implementation.

**Contrast with modern engines:**
- Modern engines use allocators/arenas with size-based bucketing, fragmentation tracking, and lifetime scopes
- C+20/Rust enforce safety; 1990s C required programmer discipline
- This design is "open for modification but closed for extension" in the literal sense—you modify the source, not subclass

**Game Engine Implication:**
The presence of this module suggests the ROTT audio subsystem was designed to be **portable and pluggable**: the host game retained full control over where and how audio buffers lived in memory. Useful if the game had its own memory management (likely true for ROTT, which managed actors, BSP, sprites, etc.).

## Potential Issues

1. **No Double-Free Detection**: `USRHOOKS_FreeMem` checks for NULL but not for invalid or already-freed pointers. A freed pointer passed a second time results in undefined behavior (memory corruption).

2. **Silent Alignment Non-Compliance**: The code does not enforce or verify dword alignment; it merely asserts "the caller assumes this." If `malloc()` doesn't guarantee alignment on the target platform, silent corruption could occur in DMA-coupled sound hardware code.

3. **No Allocation Tracking**: If a module forgets to free memory, there's no way to detect it at the library level. Relies entirely on the host application's discipline (or lack thereof).

4. **Primitive Error Handling**: The two-state error model (`USRHOOKS_Ok` / `USRHOOKS_Error`) provides no context. A caller cannot distinguish between "malloc failed due to OOM" vs. "NULL passed to free"—both return the same error code.
