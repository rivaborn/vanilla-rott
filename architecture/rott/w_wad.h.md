# rott/w_wad.h

## File Purpose
Public interface for WAD file management in the Rise of the Triad engine. WAD files are archives containing game resources (lumps) such as sprites, textures, maps, and other data. This header declares functions to load, query, and cache lumps from disk.

## Core Responsibilities
- Initialize single or multiple WAD files from disk
- Look up lumps by name or numeric index
- Query lump metadata (length, total count)
- Read lump data into caller-supplied buffers
- Cache lumps in memory with allocation tags for lifetime management

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `numlumps` | int | global | Total number of lumps loaded across all WAD files |
| `lumpcache` | void** | global | Array of cached lump pointers, indexed by lump number |

## Key Functions / Methods

### W_InitMultipleFiles
- Signature: `void W_InitMultipleFiles(char **filenames)`
- Purpose: Load and register multiple WAD files into the lump system
- Inputs: Pointer to array of filename strings
- Outputs/Return: None
- Side effects: Populates `numlumps` and `lumpcache`; reads disk files
- Calls: Not inferable from header

### W_InitFile
- Signature: `void W_InitFile(char *filename)`
- Purpose: Load and register a single WAD file
- Inputs: Filename string
- Outputs/Return: None
- Side effects: Appends lumps to global registry; reads disk
- Calls: Not inferable from header

### W_GetNumForName / W_CheckNumForName
- Signature: `int W_GetNumForName(char *name)` / `int W_CheckNumForName(char *name)`
- Purpose: Retrieve lump index by name; `Check` variant verifies existence
- Inputs: Lump name string
- Outputs/Return: Lump index (or error code for Check variant)
- Side effects: None
- Calls: Not inferable from header

### W_CacheLumpNum / W_CacheLumpName
- Signature: `void *W_CacheLumpNum(int lump, int tag)` / `void *W_CacheLumpName(char *name, int tag)`
- Purpose: Load lump into memory and cache it with a tag for memory management
- Inputs: Lump index/name; memory tag for allocation lifetime
- Outputs/Return: Pointer to cached lump data
- Side effects: Allocates memory; updates `lumpcache`
- Calls: Not inferable from header

### W_ReadLump / W_WriteLump
- Signature: `void W_ReadLump(int lump, void *dest)` / `void W_WriteLump(int lump, void *src)`
- Purpose: Read lump data into buffer or write buffer to lump
- Inputs: Lump index; buffer pointer
- Outputs/Return: None
- Side effects: Memory read/write; possible disk I/O
- Calls: Not inferable from header

### W_LumpLength / W_NumLumps / W_GetNameForNum
- Purpose: Query metadata—lump size, total count, lump name from index
- Side effects: None
- Notes: Trivial accessors

## Control Flow Notes
This is a resource management layer called during engine initialization and throughout runtime. Lumps are typically cached on first access; the tag system likely integrates with a memory manager for cleanup. Not inferable whether caching is automatic or demand-loaded.

## External Dependencies
None; this is a self-contained interface header with no visible includes or external symbol references.
