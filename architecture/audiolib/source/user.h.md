# audiolib/source/user.h

## File Purpose
Public header file for the USER module, declaring the interface for parameter checking and text retrieval functions. Part of the Apogee audio library's user configuration system.

## Core Responsibilities
- Declare function to validate/check user-provided parameters
- Declare function to retrieve text values associated with parameters
- Provide public API for parameter and configuration handling

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### USER_CheckParameter
- Signature: `int USER_CheckParameter(const char *parameter)`
- Purpose: Check or validate a user-supplied parameter
- Inputs: Parameter string to check
- Outputs/Return: Integer result (interpretation unclear from header; likely boolean or error code)
- Side effects: Not inferable from header
- Calls: Not inferable from this file
- Notes: Const parameter suggests read-only validation

### USER_GetText
- Signature: `char *USER_GetText(const char *parameter)`
- Purpose: Retrieve text value associated with a parameter
- Inputs: Parameter name/key
- Outputs/Return: Pointer to text string (caller responsibility for lifetime unclear)
- Side effects: Not inferable from header
- Calls: Not inferable from this file
- Notes: Returns pointer; unclear if result is dynamically allocated or static

## Control Flow Notes
Not inferable from this header file. Likely part of initialization/configuration setup phase.

## External Dependencies
- Standard C library (implied by function signatures)
- No explicit includes visible in this header
