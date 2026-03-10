# audiolib/source/dpmi.h — Enhanced Analysis

## Architectural Role

This file is the **protected-mode-to-real-mode bridge** for the audio library. It enables DOS-extended code (compiled for 32-bit protected mode) to allocate conventional DOS memory, lock it against paging, and invoke real-mode interrupt handlers—all critical for DMA-based sound card I/O. Without DPMI, audio drivers (GUS, Sound Blaster) cannot access the real-mode memory and interrupt vectors required by ISA-era hardware. It represents the boundary between the application's protected-mode execution environment and hardware-level real-mode constraints.

## Key Cross-References

### Incoming (who depends on this file)
- **GUS/MIDI subsystems** (`gusmidi2.c`, `gusmidi2.h`, `irq.c`): Call `DPMI_GetDOSMemory` to allocate conventional RAM for MIDI voice buffers and IRQ handlers
- **Sound Blaster driver** (inferred from audio library structure): Uses `DPMI_LockMemory` before DMA operations to pin buffers against paging
- Audio init/shutdown paths throughout `audiolib/source/`: Allocate and free DOS memory during setup

### Outgoing (what this file depends on)
- **Watcom C compiler**: `pragma aux` directive (vendor-specific inline assembly syntax)
- **DOS extender (DOS/4GW runtime)**: Provides DPMI services via `int 31h` interrupt
- **x86 CPU**: Hardware DPMI interrupt and register operations

## Design Patterns & Rationale

1. **Protected-mode abstraction layer**: Wraps raw x86 DPMI calls in C function declarations, hiding assembly details from callers.
2. **pragma aux inline assembly** (Watcom-specific): Direct code generation for DPMI int 31h calls; avoids the overhead and fragility of external assembly modules.
3. **Memory locking pair** (Lock/Unlock macros): Prevents the DOS extender's memory manager from paging buffers during DMA; macros simplify usage (`DPMI_Lock(variable)` vs. `DPMI_LockMemory(&variable, sizeof(variable))`).
4. **Register-centric calling convention**: `dpmi_regs` struct mirrors x86 ABI exactly; enables register state setup before real-mode calls and inspection after return.
5. **Descriptor-based memory handles**: DOS memory is managed via selector/descriptor handles (returned by `DPMI_GetDOSMemory`), isolating high-level code from x86 segmented memory details.

**Rationale**: In DOS-extended environments, protected-mode code is isolated from real-mode memory and I/O. DPMI is the official DOS-extender API for crossing that boundary. Watcom's `pragma aux` allows inline DPMI calls without external assembly, reducing build complexity and improving portability across linkers.

## Data Flow Through This File

1. **Audio subsystem initialization**:
   - Calls `DPMI_GetDOSMemory(buffer, &desc, size)` to allocate DMA-safe conventional memory
   - Locks the buffer with `DPMI_LockMemory(buffer, size)` to prevent paging
   - Stores descriptor handle for later deallocation

2. **Real-mode interrupt vector setup** (ISR hookup):
   - Calls `DPMI_GetRealModeVector(vector_num)` to read original interrupt vector
   - Saves original for unhook on shutdown
   - May call `DPMI_SetRealModeVector(vector_num, new_handler)` to install ISR

3. **Real-mode function calls** (e.g., DOS memory allocation):
   - Caller populates `dpmi_regs` with register state (EAX = function code, EBX = params, etc.)
   - Calls `DPMI_CallRealModeFunction(&regs)` 
   - DPMI switches CPU to real mode, executes int 31h, returns with updated registers
   - Caller checks return code and extracts results from `regs.EAX`, `regs.EDX`, etc.

4. **DMA operation execution**:
   - Audio interrupt handler reads/writes to the locked buffer
   - DMA controller accesses physical RAM directly; no paging can occur because pages are pinned

5. **Cleanup**:
   - `DPMI_UnlockMemory()` releases page pins
   - `DPMI_FreeDOSMemory()` deallocates conventional memory block

## Learning Notes

### Idiomatic to DOS-extended / ISA-era sound cards:
- **Conventional memory requirement**: ISA sound cards with DMA controllers cannot see extended memory; they see only the first 1 MB of physical RAM. Allocation must occur in "DOS memory" (real-mode address space).
- **Page pinning**: Modern paged virtual memory didn't exist in classic DOS. When DOS extenders added paging, they had to provide a way to prevent paging—`DPMI_LockMemory` fills this role.
- **Segment:offset addressing**: x86 real mode uses 16-bit segment + 16-bit offset (20-bit effective address). Protected-mode code uses linear 32-bit addresses and selectors; DPMI translates between them.
- **Register-state snapshots**: Calling real-mode code from protected mode requires saving full x86 CPU state (all registers, segment registers, flags) and restoring it afterward.

### How modern engines differ:
- Modern sound cards use PCI or USB, have their own memory controllers, and don't require conventional RAM allocation.
- Memory locking is now done via OS APIs (e.g., `mlock`, `VirtualLock` on Windows) with better error handling.
- x86 real-mode code is long obsolete; 64-bit protected mode is the norm.
- Inline assembly syntax varies by compiler; `pragma aux` is ancient Watcom-ism.

### Connections to game engine concepts:
- **Hardware abstraction layer (HAL)**: DPMI is the HAL for DOS-extended code; it hides CPU mode switching and address space translation.
- **Driver I/O boundary**: Similar to how modern drivers cross the kernel/user boundary (syscalls), DPMI crosses the protected-mode/real-mode boundary.
- **Resource lifetimes**: Memory allocation/deallocation patterns here prefigure modern resource RAII idioms.

## Potential Issues

1. **Silent failures in pragma aux blocks**: The DPMI int 31h calls check the carry flag (`jc DPMI_Exit`) but only zero EAX on success. Callers *must* check the return code; no automatic exception or logging.

2. **Register struct brittle to compiler changes**: If the compiler changes struct alignment, padding, or register order, the inline assembly will silently produce incorrect calls. There's no compile-time verification that the struct layout matches x86 ABI.

3. **Incomplete implementations**: `DPMI_LockMemory`, `DPMI_UnlockMemory`, `DPMI_LockMemoryRegion`, `DPMI_UnlockMemoryRegion`, `DPMI_GetRealModeVector`, `DPMI_SetRealModeVector`, and `DPMI_CallRealModeFunction` are declared but not defined in this header—implementations are presumably in a `.c` file (not visible here). Callers cannot inline these, limiting optimization.

4. **No bounds checking**: `DPMI_GetDOSMemory` output pointers (`ptr`, `descriptor`) are written without validation; garbage output is possible if DPMI fails silently.

5. **Portability**: Watcom `pragma aux` syntax is not portable to GCC, MSVC, or Clang without conditional compilation or alternative implementations.
