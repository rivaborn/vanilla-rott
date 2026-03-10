# rott/mouse.h — Enhanced Analysis

## Status: Unable to Complete Second-Pass Analysis

I cannot produce a meaningful enhanced analysis due to critical missing information:

### Blocking Issues

1. **Incomplete File Content**: The provided `rott/mouse.h` contains only the GPL v2 license header (19 lines). The actual type definitions, function declarations, and data structures are not included.

2. **Failed Architecture Context**: The architecture context load returned `Error: Reached max turns (1)`, meaning I don't have the subsystem-level overview needed for cross-cutting analysis.

3. **Incomplete Cross-Reference Data**: The cross-reference excerpt provided shows functions from audio, actor, and network subsystems, but contains no mouse-related functions. This suggests either:
   - Mouse functions are not in the cross-reference index yet, or
   - The excerpt was truncated before reaching the `M` section

### What I Need to Proceed

To produce a proper second-pass analysis with architectural role, cross-references, and data flows, please provide:

1. **Complete `rott/mouse.h` contents** (all function declarations, typedefs, globals)
2. **Complete architecture context** (subsystem overview, dependencies)
3. **Complete cross-reference index** (including entries for `mouse.h` functions and callers)
4. Optionally: the corresponding `rott/mouse.c` implementation file to understand how functions are actually used

Would you like me to read the complete `rott/mouse.h` file directly from the filesystem, or do you have the missing context data?
