# rott/develop.h — Enhanced Analysis

## Architectural Role

This header is the **global build-time configuration hub** for Rise of the Triad, serving as a central point where developers toggle entire feature families (debug modes, cheats, test subsystems, game variants) during compilation. Rather than scattered conditionals throughout the codebase, all compile-time decisions flow through this single configuration file, making it easy to generate different binary variants (Shareware, Super ROTT, site-licensed versions) without code changes. The mutual-exclusion comment patterns suggest this was used to produce distinct commercial SKUs in the mid-1990s before CI/CD automation.

## Key Cross-References

### Incoming (who depends on this file)
- **Broadly included**: Every translation unit that includes headers like `rt_main.h`, `rt_debug.h`, `rt_game.h`, `rt_playr.h`, `rt_net.h` (directly or transitively) gets these defines
- **Debug subsystem**: `rt_debug.c` and files it interfaces with (e.g., `CheckDebug`) rely on DEBUG, DEVELOPMENT flags to gate breakpoint code and diagnostic output
- **Player/cheat system**: `rt_playr.c` uses WEAPONCHEAT flag; weapon selection and behavior depends on this compile-time choice
- **Battle system**: Files in battle subsystem check BATTLECHECK, BATTLEINFO, SYNCCHECK for logging/validation
- **Network subsystem**: `rt_net.c` likely uses SYNCCHECK, LOADSAVETEST to gate network debugging
- **Audio subsystem**: SOUNDTEST likely gates audio diagnostics (implied by presence of SOUNDTEST define)
- **Game initialization**: `rt_game.c` and `rt_main.c` use SHAREWARE, SUPERROTT, SITELICENSE flags to select game content, story branches, or feature sets

### Outgoing (what this file depends on)
- **Global `programlocation` variable**: External global (likely `extern int programlocation` defined in `rt_main.c` or similar) written by `wami(val)` macro when WHEREAMI=1
- **No library dependencies**: Pure preprocessor—no #includes, no external symbols beyond programlocation

## Design Patterns & Rationale

**Compile-time Feature Flagging (1990s pattern)**
- All flags are preprocessor `#define`, not runtime variables. This avoids runtime overhead (branches, memory) and produces smaller, faster binaries—critical for 1994 DOS/Windows constraints.
- The `NOMEMCHECK` define suggests memory checking was a development-only tool (likely removed from production builds).

**Binary Variant Multiplexing**
- Three mutually exclusive game variants (SHAREWARE, SUPERROTT, SITELICENSE) allow a single codebase to produce three distinct commercial products. Each variant likely:
  - Includes/excludes episodes, weapons, or story content
  - Sets different limits (level cap, time limits, etc.)
  - Used different packaging/distribution channels

**Debug Instrumentation Macro Pattern**
- `wami(val)` / `waminot()` demonstrates a lightweight execution-location tracker for era-appropriate debugging (breakpoint hooks). Modern engines use stack traces and debuggers; 1994 often required manual instrumentation points.

**Rationale & Tradeoffs**
- **Advantage**: Single binary per variant, compile-time optimization, zero runtime overhead for production builds
- **Disadvantage**: No runtime reconfiguration (can't toggle debug flags without rebuild); mutual-exclusion rules are social (comments) not enforced; test flags are off by default (must rebuild to test)

## Data Flow Through This File

```
Compile Phase:
  develop.h defines → Preprocessor expands in every #include chain
                   → Code conditionally compiles (e.g., #if DEBUG ... #endif)
                   → Binary flavor (Shareware/Super ROTT) determined

Runtime (WHEREAMI only):
  wami(val) macro → programlocation = val (global variable)
                 → Used by debugger/developer to set breakpoints
```

All data flow is **compile-time only**, except the optional wami() → programlocation channel (disabled by default; WHEREAMI=0).

## Learning Notes

**Idiomatic to 1990s Game Engines**
- This pattern (centralized #define config) was standard before CMake, Bazel, or build-time argument passing. Modern engines use:
  - CMake options (`-DDEBUG=ON`)
  - Environment variables or `.env` files
  - Runtime config files (JSON, TOML)
  - Logging frameworks that respect environment-based log levels

**Execution Tracking (wami macro)**
- Shows developers relied on manual instrumentation before stack traces were standard. A developer would insert `wami(42)` in suspicious functions, set `programlocation == 42` as a debugger conditional breakpoint, then step through. Modern equivalent: stack traces, print debugging, or actual debuggers.

**Multi-SKU Product Management**
- The three exclusive game variants reflect physical retail: one code tree shipped as Shareware (limited), Super ROTT (full), and Site License (enterprise). Each binary was built once, tested, shipped. Compare to modern SaaS (feature flags, A/B tests, canary deployments).

**Missing Modern Safeguards**
- No compile-time static checks for mutual exclusion (e.g., `_Static_assert(SHAREWARE + SUPERROTT + SITELICENSE == 1)`)
- No default sanity checks at init time (e.g., `if (SHAREWARE && WEAPONCHEAT) error("incompatible")`)

## Potential Issues

1. **Unenforced Mutual Exclusion**: Comments say "Make sure only one...are on at one time" for (DELUXE/LOWCOST) and game variants, but nothing prevents a developer from compiling with SHAREWARE=1 AND SUPERROTT=1 simultaneously. This could cause feature duplication, branching logic confusion, or silent failures.

2. **WHEREAMI Default Off**: The location-tracking macro is disabled by default (WHEREAMI=0). If a developer needs it for debugging, they must rebuild. Modern alternative: conditional logging at runtime (env-var based).

3. **Beta/Release Comments Not Enforced**: Comments on BATTLECHECK and BATTLEINFO say "should be off for release, on for beta"—but nothing prevents shipping a beta build. A post-compile validation step would help.

4. **No Generated Config Header**: Best practice (modern CMake) would auto-generate a `config.h` from build flags, reducing human error. Hardcoding defines here means manual edits for each build variant.
