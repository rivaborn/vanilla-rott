# rottcom/rottser/st_cfg.h

## File Purpose

A header file declaring the setup configuration reading interface for the ROTT game engine. Provides a single entry point to load configuration data during initialization.

## Core Responsibilities

- Declare the `ReadSetup()` function for configuration initialization
- Define the public interface for setup/configuration module

## Key Types / Data Structures

None.

## Global / File-Static State

None.

## Key Functions / Methods

### ReadSetup
- Signature: `void ReadSetup(void)`
- Purpose: Load and process game setup/configuration data
- Inputs: None (reads from external sources, likely config files)
- Outputs/Return: None (void)
- Side effects: Likely modifies global game configuration state
- Calls: Not inferable from this file
- Notes: Implementation defined elsewhere; typical initialization-phase function

## Control Flow Notes

Called during engine initialization to load configuration settings before main game loop begins. No parameters suggest it operates on implicit global state or reads from standard configuration file locations.

## External Dependencies

- None visible (no includes)
- Implementation file must be located in `rottcom/rottser/` or linked module
