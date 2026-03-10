# rottcom/rottser/sermodem.h

## File Purpose
Header file declaring the modem communication interface for the ROTT network subsystem. Provides function prototypes and external configuration variables for Hayes-compatible modem control (initialization, dialing, answering, hangup).

## Core Responsibilities
- Declare modem command transmission and response reception functions
- Declare modem initialization, dial, and answer control functions
- Declare modem hangup function
- Export modem configuration strings (init, dial, hangup) and mode parameters

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `initstring` | `char[100]` | extern | Modem initialization command string (e.g., Hayes AT commands) |
| `dialstring` | `char[60]` | extern | Phone number/dial string to transmit |
| `pulse` | `boolean` | extern | Flag: use pulse dialing (true) vs. tone dialing (false) |
| `hangupstring` | `char[60]` | extern | Modem hangup/disconnect command string |

## Key Functions / Methods

### ModemCommand
- Signature: `void ModemCommand(char *str)`
- Purpose: Send a command string to the modem
- Inputs: `str` – command string to transmit
- Outputs/Return: None
- Side effects: I/O to modem port
- Calls: (not visible from header)
- Notes: Implementation handles AT command framing

### ModemResponse
- Signature: `int ModemResponse(char *resp)`
- Purpose: Read and parse modem response
- Inputs: `resp` – buffer to store response
- Outputs/Return: Integer status code
- Side effects: Reads from modem port
- Calls: (not visible from header)
- Notes: Likely returns success/failure or response code

### InitModem
- Signature: `int InitModem(void)`
- Purpose: Initialize modem with stored configuration
- Inputs: None
- Outputs/Return: Integer status (success/error code)
- Side effects: Configures modem hardware via port I/O
- Calls: (not visible from header)
- Notes: Uses `initstring` extern

### Dial
- Signature: `int Dial(void)`
- Purpose: Initiate outbound dial
- Inputs: None
- Outputs/Return: Integer status
- Side effects: Sends dial command to modem, triggers connection
- Calls: (not visible from header)
- Notes: Uses `dialstring` and `pulse` settings

### Answer
- Signature: `int Answer(void)`
- Purpose: Answer an incoming call
- Inputs: None
- Outputs/Return: Integer status
- Side effects: Sets modem to answer mode
- Calls: (not visible from header)

### hangup_modem
- Signature: `void hangup_modem(void)`
- Purpose: Disconnect/hangup modem connection
- Inputs: None
- Outputs/Return: None
- Side effects: Sends hangup command to modem
- Calls: (not visible from header)
- Notes: Uses `hangupstring` extern

## Control Flow Notes
Header only; defines the modem subsystem interface. Implementation would be in corresponding `.c` file. Functions are likely called during network initialization, connection establishment, and teardown phases of the multiplayer/network game setup.

## External Dependencies
- `global.h` – defines `boolean`, `char`, and port I/O macros (`INPUT`, `OUTPUT`)
