# rott/usrhooks.h — Enhanced Analysis

## Architectural Role

This header defines a **memory allocation abstraction layer** used throughout the ROTT engine to decouple libraries from direct `malloc`/`free` calls. It implements a **dependency injection pattern** for memory management, allowing the host application (the game) to control how and where memory is allocated—critical for 1990s systems with memory constraints, DOS extenders, or custom heaps. Any subsystem needing dynamic allocation (audio libraries, networking, cinematic data) must route through these hooks rather than using standard C runtime allocators.

## Key Cross-References

### Incoming (who depends on this file)
The provided cross-reference index cuts off at function names starting with "C" and does not include entries for `USRHOOKS_*` functions. However, based on the codebase structure and the module header comment mentioning this interface is for "the library," callers would include:
- **audiolib/** subsystem (`adlibfx.c`, `blaster.c`, `gus.c`, `awe32.c`, etc.)—these sound drivers allocate voice buffers and instrument data
- **Network libraries** (rottipx, rottser packages)—likely allocate packet buffers
- Any other third-party or portable code included in the engine

### Outgoing (what this file depends on)
- None—this is a pure declaration header with no dependencies
- Implementation of `USRHOOKS_GetMem` and `USRHOOKS_FreeMem` defined in `usrhooks.c` (referenced in module header)

## Design Patterns & Rationale

**Dependency Injection for Resource Management**: Rather than libraries calling `malloc`/`free` directly, they call through these hook functions. The host app provides implementations that route to custom allocators, memory pools, or restricted heaps.

**Error-Code Return Model**: Functions return `int` status codes (not the allocated pointer directly) to allow distinguishing allocation failure (`USRHOOKS_Error`) from success (`USRHOOKS_Ok`). The `USRHOOKS_Warning` status suggests the library supported partial success or fallback behavior (e.g., allocation succeeded but not at optimal location).

**Rationale (1994-1995 context)**:
- DOS/Protected Mode required careful heap management (real mode vs. extended memory)
- Multiple allocators might be active (game heap, audio heap, network heap)
- External libraries needed to respect host constraints without being rewritten
- No C++ templates or allocator objects; callback functions were the language feature available

## Data Flow Through This File

1. **Initialization**: Host application implements `USRHOOKS_GetMem` / `USRHOOKS_FreeMem` (in `usrhooks.c`)
2. **Library Allocation**: Audio library (e.g., `adlibfx.c`) calls `USRHOOKS_GetMem(&buffer, 4096)` instead of `malloc(4096)`
3. **Host Decision**: Host implementation (custom allocator) receives request and either:
   - Allocates from game heap, writes pointer to `*ptr`, returns `USRHOOKS_Ok`
   - Returns `USRHOOKS_Error` if allocation fails
4. **Library Use**: Library proceeds with allocated buffer
5. **Cleanup**: Library calls `USRHOOKS_FreeMem(ptr)` when done

## Learning Notes

**Idiomatic to 1990s Game Development**:
- Manual resource management via explicit allocation/deallocation hooks
- No RAII (Resource Acquisition Is Initialization) pattern; relies on discipline
- Callback-based abstraction instead of virtual functions or templates

**How Modern Engines Do It Differently**:
- C++ custom allocators via templates (`std::allocator<T>`)
- Memory pools and arenas with structured lifetimes
- Debug tracking and leak detection built into allocators
- Automatic cleanup via smart pointers

**Engine Architecture Insight**:
- ROTT was built to be portable: the same library code could run on DOS, Windows, or other platforms by swapping the hook implementation
- Suggests the audio, network, and cinematic subsystems were originally portable libraries licensed or ported from elsewhere

## Potential Issues

1. **Null Pointer Dereference**: No guard on `ptr` parameter in `USRHOOKS_GetMem`—a buggy or hostile hook implementation could pass NULL and cause a crash when libraries dereference `*ptr`

2. **No Size Validation**: No check that `size > 0`; calling code might request 0 bytes and get unexpected behavior

3. **Ambiguous Error Codes**: Error codes are negative integers, but if the hook mistakenly returns a success status that looks like a pointer cast, or if error handling is inconsistent, it could lead to silent failures

4. **No Memory Query**: No way to ask "how much memory is available?" before allocation—libraries must allocate and handle failure, not pre-check capacity

5. **No Alignment Guarantee**: Hook signature doesn't specify alignment requirements (16-byte, cache-line, DMA, etc.), leaving room for mismatches if specialized memory is needed
