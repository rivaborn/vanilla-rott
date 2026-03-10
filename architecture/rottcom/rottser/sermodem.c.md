# rottcom/rottser/sermodem.c

## File Purpose

Implements modem control and AT command interface for establishing serial multiplayer connections. Handles modem initialization, dialing outbound numbers, answering incoming calls, and disconnecting via DTR control and AT command sequences.

## Core Responsibilities

- Send AT commands to modem with character-level pacing and delays
- Parse and validate modem responses (OK, RING, CONNECT)
- Initialize modem with stored initialization string
- Dial outbound connections using pulse or tone dialing
- Answer incoming calls and await connection establishment
- Disconnect by toggling DTR and issuing hangup command

## Key Types / Data Structures

None.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| initstring | char[100] | global | Modem initialization AT command string |
| dialstring | char[60] | global | Phone number to dial |
| pulse | boolean | global | Dialing mode (true = pulse, false = tone) |
| hangupstring | char[60] | global | Modem hangup AT command string |

## Key Functions / Methods

### hangup_modem
- **Signature:** `void hangup_modem(void)`
- **Purpose:** Disconnect modem by dropping DTR, sending escape, and AT hangup command.
- **Inputs:** None (uses global `hangupstring`)
- **Outputs/Return:** None
- **Side effects:** Modifies UART MCR register; drains input buffer; prints to stdout
- **Calls:** `INPUT()`, `OUTPUT()`, `delay()`, `ModemCommand()`, `read_byte()`
- **Notes:** Falls back to hardcoded "ATH0" if `hangupstring` is empty; 1250ms delays for modem timing.

### ModemCommand
- **Signature:** `void ModemCommand(char *str)`
- **Purpose:** Send AT command to modem byte-by-byte with 100ms delays between characters.
- **Inputs:** `str` — null-terminated AT command string
- **Outputs/Return:** None
- **Side effects:** Writes to modem via `write_buffer()`; prints command to stdout
- **Calls:** `strlen()`, `printf()`, `write_buffer()`, `delay()`
- **Notes:** Appends carriage return; 100ms delay between each character to allow modem processing.

### ModemResponse
- **Signature:** `int ModemResponse(char *resp)`
- **Purpose:** Wait for modem response matching expected prefix; check for user abort (ESC key).
- **Inputs:** `resp` — expected response string prefix (e.g., "OK", "RING", "CONNECT")
- **Outputs/Return:** TRUE if response matched; FALSE if ESC pressed
- **Side effects:** Reads serial input; accumulates into 80-byte buffer; prints to stdout; polls keyboard
- **Calls:** `bioskey()`, `read_byte()`, `strncmp()`, `printf()`
- **Notes:** Case-sensitive; filters non-printable chars; newline or buffer-full (79 chars) ends accumulation; loops until response matches.

### InitModem
- **Signature:** `int InitModem(void)`
- **Purpose:** Send initialization string to modem and verify "OK" response.
- **Inputs:** None (uses global `initstring`)
- **Outputs/Return:** TRUE if initialized or no init string; FALSE if no "OK" response
- **Side effects:** Calls `ModemCommand()`, `ModemResponse()`
- **Calls:** `ModemCommand()`, `ModemResponse()`
- **Notes:** Skips if `initstring[0] == EOS` (empty string).

### Dial
- **Signature:** `int Dial(void)`
- **Purpose:** Initialize modem and dial phone number; return CONNECT status.
- **Inputs:** None (uses globals `dialstring`, `pulse`, sets `usemodem`)
- **Outputs/Return:** TRUE if "CONNECT" received; FALSE otherwise
- **Side effects:** Sets `usemodem = true`; formats dial command via `sprintf()`
- **Calls:** `InitModem()`, `sprintf()`, `ModemCommand()`, `ModemResponse()`, `printf()`
- **Notes:** Formats "ATDP" (pulse) or "ATDT" (tone) based on `pulse` flag.

### Answer
- **Signature:** `int Answer(void)`
- **Purpose:** Initialize modem and answer incoming call; return CONNECT status.
- **Inputs:** None (sets `usemodem`)
- **Outputs/Return:** TRUE if answered with "CONNECT"; FALSE if no "RING"
- **Side effects:** Sets `usemodem = true`
- **Calls:** `InitModem()`, `ModemResponse()`, `ModemCommand()`, `printf()`
- **Notes:** Waits for "RING", then sends "ATA" (answer).

## Control Flow Notes

Called during game initialization/shutdown for serial multiplayer:
- **Dial()** / **Answer()** establish connection during setup phase
- **hangup_modem()** tears down connection at shutdown
- Bridges game config (dialstring, initstring) to low-level serial I/O (UART, read_byte, write_buffer)

## External Dependencies

- **System headers:** `<time.h>`, `<stdio.h>`, `<string.h>`, `<bios.h>` (DOS era)
- **Local headers:** global.h, serial.h, port.h, sermodem.h, sersetup.h
- **Defined elsewhere:**
  - `INPUT()`, `OUTPUT()` macros (UART I/O via inp/outp)
  - `uart`, `MODEM_CONTROL_REGISTER`, `MCR_DTR` constants
  - `delay()` function
  - `read_byte()`, `write_buffer()` (serial drivers)
  - `bioskey()` (DOS BIOS keyboard)
  - `usemodem` flag
