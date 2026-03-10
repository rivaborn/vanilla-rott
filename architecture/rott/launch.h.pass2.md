# rott/launch.h — Enhanced Analysis

## Architectural Role
This header implements a low-level DOS driver abstraction layer for ProAudio Spectrum (PAS) audio hardware, functioning as the **hardware detection and invocation interface** within ROTT's legacy audio subsystem. It bridges the gap between the game engine's audio initialization (likely in `rt_draw.c` or audio lib startup code) and x86 real-mode driver protocols via DOS interrupt conventions. The three exported functions form a bootstrapping sequence: detect hardware → retrieve driver function table → invoke driver operations.

## Key Cross-References
### Incoming (who depends on this file)
- **Audio subsystem initialization**: Likely called from `rt_draw.c::BuildTables()` or equivalent audio setup function during engine startup (not explicitly visible in cross-reference excerpt, but inferable from first-pass "occurs during engine startup" note)
- **Referenced by**: Header is included in audio library setup code (BLASTER_*, AL_*, ADLIBFX_* subsystems visible in cross-ref suggest parallel driver abstractions)

### Outgoing (what this file depends on)
- **DOS BIOS layer**: MV_* interrupt codes (0xbc** range) communicate directly with PAS BIOS at interrupt 0x2f
- **No C library dependencies**: Pure DOS driver protocol constants; no stdio, stdlib, or standard headers
- **x86 register calling conventions**: Relies on compiler's support for far pointers and register parameter passing

## Design Patterns & Rationale
**Function Pointer Table (MVFunc)**: Delegates audio operations to driver-provided functions rather than hard-coded hardware I/O. Enables:
- Runtime driver capability discovery (not all PAS cards support all operations)
- Safe indirect calls without knowing function addresses ahead of time
- Extensibility for future PAS hardware variants

**Register-based API (PAS_CallMVFunction)**: Mimics DOS era driver calling conventions where bx, cx, dx carry command parameters. Avoids stack overhead; direct hardware control via registers matches BIOS interrupt architecture.

**Stateless detection (PAS_CheckForDriver)**: Returns pass/fail; no retained driver state. Supports multiple init attempts or recovery from driver load failure.

## Data Flow Through This File
```
Engine Startup
  ↓
PAS_CheckForDriver()  [Return: presence boolean]
  ↓ (if present)
PAS_GetFunctionTable()  [Return: MVFunc* pointing to driver operations table]
  ↓
Game Loop / Audio Setup
  ↓
PAS_CallMVFunction(func_ptr, bx, cx, dx)  [Invoke: SetMixer, SetVolume, GetFilter, etc.]
  ↓
Hardware State (mixer routing, volume levels, FM split mode)
```

Driver functions in MVFunc never directly appear in game code—they're invoked through `PAS_CallMVFunction()` with bx/cx/dx values pre-configured by audio subsystem logic (elsewhere).

## Learning Notes
**DOS Era Conventions**: 
- `far` pointers = segmented memory model (DOS real mode); unheard of in modern systems
- MV_* interrupt codes are **hardcoded BIOS protocol numbers** from ProAudio Spectrum hardware spec
- Register-passing is faster than stack frames but requires inline asm or compiler intrinsics

**What Modern Engines Do Differently**:
- **Plugin architectures**: Instead of function tables, use shared library loading (.so, .dll) with symbol resolution
- **Abstraction layers**: HAL (Hardware Abstraction Layer) interfaces in C++ or Rust, not bare register passing
- **Zero coupling to calling conventions**: Function pointers don't encode x86 register names

**Engine-Specific Pattern**:
ROTT implements parallel driver abstractions for multiple sound cards (PAS, Blaster, AdLib, GUS via ADLIBFX/AL_/BLASTER namespaces in cross-ref). This header is **one of several hardware drivers**, suggesting a **driver registry pattern** where the engine probes for available hardware at startup and selects the first match.

## Potential Issues
- **No error propagation from driver calls**: `PAS_CallMVFunction()` returns int but caller doesn't know which operation failed or why—only generic error codes from driver
- **Assumptions about far pointer semantics**: Code assumes compiler supports far pointers; fails silently on modern 32/64-bit systems without emulation
- **No idempotency guards**: `PAS_CheckForDriver()` can be called repeatedly; if driver state is maintained somewhere, repeated calls might corrupt it (though this is unlikely for simple detection)

---

**Note on Cross-Reference Limitations**: The provided cross-reference excerpt does not list `PAS_*` function calls directly, suggesting these functions are either defined in a non-indexed file or called infrequently. The audio library subsystem (BLASTER, AL, ADLIBFX families) is well-indexed, implying this PAS layer may be a legacy fallback or optional driver variant.
