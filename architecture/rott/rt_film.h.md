# rott/rt_film.h

## File Purpose
Public interface for the film/movie playback system. Declares a single function to initiate movie playback and exposes a global variable for display center positioning during movie rendering.

## Core Responsibilities
- Declare the public movie playback entry point
- Export display center Y coordinate for screen positioning
- Provide minimal abstraction for cinematic/demo sequences

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| dc_ycenter | int | extern | Display center Y coordinate, used for vertical centering during movie playback |

## Key Functions / Methods

### PlayMovie
- Signature: `void PlayMovie(char * name)`
- Purpose: Initiate playback of a movie file by name
- Inputs: `name` – pointer to null-terminated string specifying movie filename
- Outputs/Return: void
- Side effects: Likely performs file I/O, manages video playback state, controls screen rendering, may block until playback completes
- Calls: Not inferable from this file (implementation in rt_film.c)
- Notes: Takes unvalidated char pointer; no documented constraints on movie format, file path, or naming convention. Function appears blocking rather than asynchronous.

## Control Flow Notes
Entry point for cinematic/demo sequences, likely called from menu systems or initialization code rather than per-frame game loop. Execution control is surrendered to the movie player until playback ends.

## External Dependencies
- Implementation provided by rt_film.c
- `dc_ycenter` defined elsewhere in codebase
