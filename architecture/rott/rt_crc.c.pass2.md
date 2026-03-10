# rott/rt_crc.c — Enhanced Analysis

## Architectural Role
CRC-16 checksum calculation serves as the engine's data integrity layer, supporting both save game validation and network packet synchronization in ROTT's multiplayer architecture. As a stateless utility library, it enables fast incremental checksum computation without allocation overhead—critical for a 1995 engine where streaming updates and per-packet validation were performance-sensitive operations. The table-driven algorithm is optimized for DOS/386 performance constraints of the era.

## Key Cross-References

### Incoming (who depends on this file)
- **Game state save/load**: Functions like `CP_LoadGame`/`CP_SaveGame` (rt_menu.h) likely validate serialized game state with `CalculateCRC`
- **Network synchronization**: References to `CheckForSyncCheck` (rt_net.h) and `BATTLE_CheckGameStatus` (rt_battl.h) suggest CRC validates packet integrity and game state consistency across network clients
- **Memory verification**: Include of `memcheck.h` combined with CRC suggests integration into memory debugging/validation routines (though function not visible in cross-reference)

### Outgoing (what this file depends on)
- **Standard library only**: stdio.h, stdlib.h, string.h (minimal external dependencies)
- **Type system**: References `byte` and `word` types (defined in rt_def.h, pulled via rt_crc.h)
- **No runtime calls**: Entirely self-contained; uses only static lookup table

## Design Patterns & Rationale
**Table-driven algorithm**: Pre-computed crc16tab[256] converts O(8) bit operations per byte into O(1) table lookup—a critical optimization for 1990s DOS performance. This pattern trades 512 bytes of ROM for faster per-packet validation in networked games where CRC may be calculated thousands of times per second.

**Stateless design**: Both `updatecrc()` and `CalculateCRC()` are pure functions (no static state beyond the lookup table), enabling easy composition—buffers can be checksummed incrementally across multiple calls or in batch, and the same CRC value can be validated in-engine or over the network.

**Dual interface**: `updatecrc()` (1-byte streaming) and `CalculateCRC()` (batch) provide flexibility for different call sites without code duplication at the algorithm level.

## Data Flow Through This File
1. **Entry**: Buffers arrive from file I/O (`CP_LoadGame`), network receive (`rt_net`), or memory verification
2. **Processing**: Each byte XORs with CRC low byte to index table; high byte shifts and XORs with table value
3. **Exit**: CRC-16 checksum returned as `word` for comparison against stored/transmitted checksum
4. **Use case**: Save game headers, network packets, and in-game state snapshots likely embed CRC for detect corruption

## Learning Notes
- **Idiomatic 1990s**: Polynomial-based checksums (not cryptographic hashes) were the standard for game engine integrity—fast, deterministic, small output.
- **Reverse-bit CRC-16**: The algorithm processes bits in reverse order (note `crc>>8` and `tmp & 0xff` logic), a variant optimized for hardware implementation on older platforms.
- **No initialization secret**: CRC always starts at 0; modern engines might use a seed for better collision properties, but fixed zero was simpler and sufficient for game data.
- **Contrast with modern engines**: Contemporary systems use MD5/SHA-1 for integrity; this engine uses lightweight CRC-16, reflecting memory/bandwidth constraints.

## Potential Issues
- **No buffer bounds checking**: `CalculateCRC()` trusts caller-provided `size` parameter; invalid sizes cause silent read past buffer
- **Fixed initial state**: Hardcoded checksum=0 prevents validating incomplete streams or chained operations that might benefit from non-zero seed
- **Commented driver code**: The disabled `main()` function (lines 119–145) is debug cruft that should be removed
