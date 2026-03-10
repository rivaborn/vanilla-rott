# rott/_rt_stat.h

## File Purpose
Private header file declaring functions and types for managing static objects (doors, walls, decorations, lights) in the ROTT game engine. Provides data structures for animated wall info and saved static state, plus utility macros for fire color animation and light detection.

## Core Responsibilities
- Declare functions for adding and managing static/animated static objects
- Define types for persisting static object state (saved_stat_type)
- Declare precaching function for static object sounds
- Provide animation timing constants and light detection utility

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `awallinfo_t` | struct | Stores animated wall metadata: animation timing, frame count, first lump name |
| `saved_stat_type` | struct | Complete state snapshot of a static object for save/load: position, flags, hitpoints, animation state, linked object reference |

## Global / File-Static State
None.

## Key Functions / Methods

### AddStatic
- Signature: `void AddStatic(statobj_t*)`
- Purpose: Register a static object into the game world
- Inputs: Pointer to statobj_t object
- Outputs/Return: None
- Side effects: Modifies global static object list
- Calls: (not visible—implementation elsewhere)
- Notes: Called during level load; implementation likely manages spatial data structures

### AddAnimStatic
- Signature: `void AddAnimStatic(statobj_t*)`
- Purpose: Register an animated static object into the game world
- Inputs: Pointer to statobj_t object
- Outputs/Return: None
- Side effects: Modifies global static object list; may register animation updates
- Calls: (not visible—implementation elsewhere)
- Notes: Variant of AddStatic for objects with active animations

### PreCacheStaticSounds
- Signature: `void PreCacheStaticSounds(int)`
- Purpose: Load and cache audio samples needed by static objects
- Inputs: Integer parameter (likely sound group or count; exact meaning not inferable)
- Outputs/Return: None
- Side effects: Loads sound assets into memory
- Calls: (not visible—implementation elsewhere)
- Notes: Called during initialization to avoid audio load stalls during gameplay

## Control Flow Notes
Part of the static object initialization and save/load pipeline. Objects are added during level load via AddStatic/AddAnimStatic; saved_stat_type is used to serialize state for save games. The IsLight macro is used during gameplay (likely for lighting/visibility calculations).

## External Dependencies
- `statobj_t` – object type (defined elsewhere)
- `sprites` global array – 2D sprite lookup (defined elsewhere)
- Standard C types: int, char, byte, short int, signed char
