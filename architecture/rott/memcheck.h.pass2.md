# rott/memcheck.h — Enhanced Analysis

## Architectural Role

MemCheck 3.0 is a **development-time debugging tool** that operates orthogonally to the runtime game engine architecture—it's not part of the normal control flow and can be completely eliminated via the `NOMEMCHECK` define. Rather than being called as a subsystem, it operates by intercepting standard library calls (malloc, free, strcpy) at the preprocessor level in each translation unit where it's included, providing source-level tracking without requiring linker support or separate libraries. This design reflects DOS-era constraints where runtime overhead and linker complexity were prohibitive.

## Key Cross-References

### Incoming (who depends on this file)
- **Individual .c source files** (not visible in the cross-reference excerpt) likely include this header per the comment: "should be #included AFTER any other #includes." The lack of memcheck symbols in the function definition map suggests it's used via header-only macro interception, not via exported function calls.
- No subsystem directly calls MemCheck API functions as part of normal game flow—integration is compile-time/macro-based only.

### Outgoing (what this file depends on)
- **Compiler runtime libraries**: malloc.h, alloc.h, string.h (all compiler-specific variants for different DOS compilers)
- **Standard C headers**: stdio.h, stdarg.h (for error reporting)
- **Compiler intrinsics**: Detects and disables compiler-specific memory functions (like `_fmalloc`, `_nmalloc` under MSC/Borland)
- **DOS system knowledge**: Stack segment addresses (STACKTOP, STACKEND), memory models (LCODE, LDATA, far/near addressing)

## Design Patterns & Rationale

1. **Macro Interception at Preprocessor Time**
   - Instead of runtime dispatch or linker substitution, every `malloc(size)` becomes `_INTERCEPT(malloc(size))` which expands to location tracking + renamed RTL call.
   - This was the gold standard for legacy C debugging because: (a) zero runtime overhead when disabled, (b) exact source line tracking without frame pointers, (c) works across different compilers with their own calling conventions.

2. **Compiler Abstraction Layer (_CCDEFS_H_)**
   - Detects 7+ DOS-era compilers (MSC 5.x–8.x, Borland TCC/PowerPack, Watcom, Intel Code Builder) and normalizes memory model macros.
   - Each compiler had incompatible built-in functions (e.g., MSC's `_halloc` vs. Borland's `farmalloc`), so MemCheck must know which symbols to intercept and rename.

3. **Bidirectional Callback System**
   - **Outgoing callbacks** (ERF, TRACKF, CHECKF, GLOBALF) allow user code to hook into MemCheck events.
   - **Incoming callbacks** (like disk Rocket allocator functions) allow MemCheck to delegate allocation to custom providers.
   - This flexibility was essential for embedded/real-time systems (e.g., game engines managing memory pools).

4. **Dual-Mode Design (NOMEMCHECK vs. Active)**
   - When `NOMEMCHECK` is defined, all macro calls evaporate to direct RTL calls—zero overhead, no linker dependencies.
   - This was vastly superior to `#ifdef DEBUG` guards around the include because it doesn't force users to wrap each API call (`mc_startcheck()`, etc.) separately.

## Data Flow Through This File

1. **Initialization Phase**
   - Developer includes `memcheck.h` after standard library headers.
   - Preprocessor defines all interception macros and callback declarations.
   - `mc_startcheck()` called explicitly or via AutoInit flag → initializes B-tree database, reads config file if present.

2. **Runtime Interception**
   - Every `malloc()` → `_INTERCEPT(malloc())` → calls `_mcsl()` (location push) + renamed `malloc_mc()` → allocator adds MEMREC to B-tree.
   - TRACKF callback fires on allocation/deallocation.
   - `strcpy()`, `memcpy()` → `mc_check_transfer()` → validates source/dest pointers and bounds → CHECKF callback if registered.

3. **Error Detection**
   - Illegal write to freed buffer detected at next `mc_check()` call (magic byte corruption).
   - Stack overflow detected by `mc_stack_trace()` or exception handler callback.
   - Memory leak detected at `mc_endcheck()` (allocations never freed).

4. **Shutdown**
   - `mc_endcheck()` → reports all unfreed MEMREC entries → ENDF callback → returns error flags bitfield.

## Learning Notes

**What a developer studying this file learns:**

- **Legacy DOS debugging philosophy**: Before modern tools (Valgrind, AddressSanitizer, MSan), this was the state-of-the-art for memory debugging. It demonstrates:
  - Preprocessor-based instrumentation (now replaced by compiler instrumentation flags like `-fsanitize=address`)
  - Callback-based extensibility (now replaced by static analysis and runtime sanitizers)
  - Compiler-specific quirks and workarounds (obsolete in modern cross-platform development)

- **Cross-compiler abstraction patterns**: The `_CCDEFS_H_` section is a masterclass in handling fragmented DOS toolchain ecosystems. Modern developers would use CMake or autoconf, but this is pure preprocessor discipline.

- **Far/near pointer semantics**: The repeated `_MCFAR` macro and far/near address abstractions are alien to modern developers—they reflect real hardware segmentation (real mode vs. protected mode), now purely historical.

- **Idiomatic differences**: Modern engines use:
  - **Custom allocators** (arena, pool, bump allocators) instead of malloc wrappers.
  - **Address Sanitizer** (compiler-based instrumentation) instead of macro magic.
  - **Memory tagging** (Arm MTE) and OS page guards instead of magic byte corruption detection.

## Potential Issues

1. **Inclusion Order Fragility**
   - Documentation warns memcheck.h "MUST NOT come before any prototypes of routines that MemCheck intercepts."
   - If a file includes `<string.h>` before `<memcheck.h>`, the `strcpy` prototype may not be intercepted—silent tracking failure. No compile-time check for this.

2. **Macro Namespace Pollution**
   - Redefines common symbols like `malloc`, `free`, `strcpy` globally. If a third-party header declares these as inline functions or uses name-mangling, the interception may break.

3. **Callback Thread Safety**
   - TRACKF, CHECKF callbacks are invoked from every allocation context. No documentation of thread-safety guarantees. In multithreaded code, race conditions on global state (MC_Settings, MC_TrackF pointer) are possible.

4. **Exception Frame Parsing**
   - MCEXCEPTINFO is defined in variant form (16-bit vs. 32-bit registers). Manual register frame parsing is fragile and compiler-version-dependent.

5. **Obsolescence**
   - This tool assumes DOS/DPMI environment. Inclusion in a modern Win32/Linux cross-platform codebase (which vanilla-rott appears to be) suggests it's likely dead code or used only during original development.
