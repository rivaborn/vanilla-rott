# rott/w_wad.c — Enhanced Analysis

## Architectural Role

This file implements the **asset abstraction layer** for ROTT, providing a unified virtual filesystem interface between game code and disk storage. It sits at a critical bridge: below the asset-specific loaders (sprite system, map loader, configuration parser) and above the zone memory manager. By decoupling asset access from I/O, it enables the game to treat a fragmented multi-file WAD archive as a single addressable namespace, and integrates lazy-load caching directly into the zone memory lifecycle.

## Key Cross-References

### Incoming (who depends on this file)
- **Asset loaders** across the engine call `W_CacheLumpNum()` or `W_CacheLumpName()` on-demand:
  - Sprite/texture system (caches artwork lumps)
  - Map data loader (reads level geometry)
  - Configuration/script parsers (loads `.rts` scripts and `.cfg` data)
  - Sound/music system (may load sound lumps, given `W_CheckWADIntegrity()` skips if `SOUNDSETUP` is true)
- **Game initialization** (`rt_main.c` implied) calls `W_InitMultipleFiles()` or `W_InitFile()` at startup
- **Debug/cheat system** (`rt_debug.c` implied) may read lumps directly via `W_ReadLump()`

### Outgoing (what this file depends on)
- **Zone memory manager** (`z_zone.h`): `Z_Malloc()`, `Z_Realloc()`, `Z_ChangeTag()` — all lump caching is zone-allocated with caller-supplied tags, allowing unified lifetime management
- **Utilities** (`rt_util.h`): `SafeMalloc()`, `ExtractFileBase()` — lump name extraction and safe allocation
- **CRC subsystem** (`rt_crc.h`): `CalculateCRC()` — for WAD integrity verification (copy-protection mechanism)
- **Error handling** (implicit): `Error()`, `SoftError()` for fatal and soft errors
- **Platform/config** (`develop.h`, `rt_main.h`): Feature flags like `DATACORRUPTIONTEST`, `SOUNDSETUP`, `PRECACHETEST` for optional runtime validation
- **POSIX I/O**: `open()`, `read()`, `write()`, `lseek()`, `fstat()`

## Design Patterns & Rationale

**Doom-inherited WAD Architecture:**  
Multi-lump archives with a centralized directory table, mirroring Doom's design. Single-file lumps (bare assets) are promoted to lump status using filename as name—elegant fallback for simple assets.

**Lazy-Load + Zone Integration:**  
Caching defers disk I/O to first access (`lumpcache` is null-checked before read). Zone memory tagging allows the memory manager to evict/reload lumps under pressure without explicit cache management in caller code—critical for 1990s DOS memory constraints.

**Copy Protection via CRC:**  
`W_CheckWADIntegrity()` computes a checksum over the entire lumpinfo table at init; mismatch triggers a warning (Apogee branding + plea to purchase). Modern engines skip this, but it's characteristic of 90s shareware protection.

**Optimized Name Lookup:**  
Lump names are 8 bytes; comparison is done as two 32-bit word comparisons instead of string-by-string, avoiding strlen() overhead. Trade-off: fixed name length, but acceptable given Doom convention.

**Last-Loaded Wins Override:**  
Duplicate lump names are allowed; linear forward scan means later files shadow earlier ones. Supports patching without modifying base WADs.

## Data Flow Through This File

```
[Game Start]
    ↓
W_InitMultipleFiles([base.wad, patch.wad, ...])
    ↓ (for each file)
W_AddFile() → open file → parse header/lumps → Z_Realloc(lumpinfo) 
    ↓
[lumpinfo fully built; file handles held open]
    ↓
[Game Runtime]
    ↓
Sprite/Map/Config loaders call W_CacheLumpName("SPRITES", TAG_GRAPHICS)
    ↓
W_CheckNumForName() → linear scan of lumpinfo (O(n))
    ↓
W_CacheLumpNum() → lumpcache[i] already set? Yes→return; No→
    ↓
Z_Malloc(size, tag) → allocate zone heap
    ↓
W_ReadLump() → lseek(handle, lumpinfo[i].position) + read()
    ↓
[if DATACORRUPTIONTEST] compute CRC, store alongside data
    ↓
return pointer to zone-allocated data
    ↓
[Zone manager later frees when tag priority dictates]
```

## Learning Notes

**Idiomatic 90s Game Architecture:**  
This file exemplifies the pre-streaming, pre-virtual-memory-VM era. Assets are chunked into "lumps" (units of work), kept in a global registry, and manually lifetime-managed via zone tagging. Modern engines use streaming, async I/O, and asset handles—but ROTT's model is simpler and transparent.

**Doom Lineage:**  
The WAD format (IWAD header, lump directory offset, 8-char names) is directly inherited from Doom. Apogee licensed or reverse-engineered this design, making ROTT's asset pipeline highly compatible with Doom modding tools.

**Copy Protection as Code:**  
The CRC integrity check is defensive copy-protection. It doesn't prevent piracy, but signals branding and discourages casual modification. Ties asset loading directly to release validation.

**Endianness Sensitivity:**  
`LONG()` macro is applied to WAD headers and lump metadata but assumes consistent byte order on read. This would be a porting hazard to big-endian systems (e.g., MIPS).

## Potential Issues

- **File handle exhaustion:** WAD file handles remain open for the entire game lifetime. On systems with low handle limits (<64), loading many patch WADs could cause `open()` to fail silently (returns -1, function exits early, no error reported).
- **Linear name lookup:** `W_CheckNumForName()` is O(n) for ~100–500 lumps. Acceptable, but a hash table would be faster and is trivial to add.
- **CRC only on init:** WAD integrity is checked once at startup (unless `SOUNDSETUP` suppresses it). Runtime memory corruption to lumpcache data is only detected if `DATACORRUPTIONTEST` is enabled and periodic checks trigger.
- **Implicit zone tag coupling:** Callers must know the correct zone tag for their lump type. No compile-time checking; mismatch leads to premature or delayed eviction.
