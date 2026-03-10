# rottcom/rottnet.c

## File Purpose
Initializes network communication infrastructure for ROTT by setting up interrupt vectors, allocating a dedicated interrupt stack, and launching the main ROTT executable with network parameters. This is a bridge between the network launcher and the game itself, handling low-level ISR setup required for IPX/serial multiplayer.

## Core Responsibilities
- Allocate and manage a private stack for network interrupt service routine execution
- Find and hook an available DOS interrupt vector (0x60–0x66 range) for network communication
- Save/restore the interrupt vector lifecycle during setup and shutdown
- Parse command-line parameters and build argument list for launching ROTT
- Implement the ROTTNET_ISR interrupt handler wrapper that switches stacks
- Configure ticstep (game tick skipping) based on game type (modem vs. network)
- Spawn the main ROTT executable via `spawnv` and wait for completion

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| rottcom_t | struct | Main network communication state (declared in rottnet.h, instantiated as global `rottcom`) |
| que_t | struct (from port.h) | Ring buffer for serial port I/O (inque, outque) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| rottcom | rottcom_t | global | Central network communication config (intnum, gametype, ticstep, consoleplayer) |
| vectorishooked | int | global | Flag: whether interrupt vector is currently installed |
| pause | boolean | global | Flag: pause before launching ROTT for debugging/verification |
| oldrottvect | void interrupt (*)() | global | Saved previous interrupt handler at the hooked vector |
| rottnet_stack | char* | static | Allocated heap buffer for ISR stack |
| rottnet_stacksegment | unsigned short | static | Real-mode segment address of stack top |
| rottnet_stackpointer | unsigned short | static | Real-mode offset address of stack top |
| old_stacksegment, old_stackpointer | unsigned short | static | Saved caller's stack pointers (per-ISR invocation) |

## Key Functions / Methods

### SetupROTTCOM
- **Signature:** `void SetupROTTCOM(void)`
- **Purpose:** Initialize the ROTTCOM structure by finding an available interrupt vector and allocating a private stack for the ISR.
- **Inputs:** None (reads global state via `GetVector()`, command-line via `CheckParm()`)
- **Outputs/Return:** None (modifies globals: `rottcom.intnum`, `rottnet_stack*`)
- **Side effects:** Allocates heap memory (via `malloc`); calls `Error()` if allocation fails.
- **Calls:** `GetVector()`, `CheckParm()` (from global.h), `malloc()`, `Error()`
- **Notes:** Uses far pointers to scan interrupt vector table in real-mode DOS memory (vector*4). If no free vector found in 0x60–0x66, defaults to 0x66 with warning. Checks for NULL or 0xcf (iret instruction) as indicators of unused vectors.

### ShutdownROTTCOM
- **Signature:** `void ShutdownROTTCOM(void)`
- **Purpose:** Restore the original interrupt handler and free the allocated ISR stack.
- **Inputs:** None (reads globals)
- **Outputs/Return:** None (modifies globals: `vectorishooked`, heap)
- **Side effects:** Calls `setvect()` to restore old interrupt handler; frees allocated stack via `free()`.
- **Calls:** `setvect()`, `free()`
- **Notes:** Safe to call even if vector was never hooked (checks `vectorishooked` flag).

### GetVector
- **Signature:** `long GetVector(void)`
- **Purpose:** Retrieve the interrupt vector number from command-line parameter `-vector` or return -1 if not specified.
- **Inputs:** Command-line arguments (`_argv`, `_argc` globals)
- **Outputs/Return:** Long integer (vector number 0x60–0x66, or -1 if not specified)
- **Side effects:** Calls `Error()` if specified vector is already hooked.
- **Calls:** `CheckParm()`, `sscanf()`, `Error()`
- **Notes:** Uses far pointer to check if vector is in use; rejects vectors with non-NULL handlers or missing iret.

### ROTTNET_ISR
- **Signature:** `void interrupt ROTTNET_ISR(void)`
- **Purpose:** Interrupt service routine wrapper that switches to a private stack before calling the real network ISR logic.
- **Inputs:** None (CPU state at interrupt time)
- **Outputs/Return:** None (interrupt return via implicit `iret`)
- **Side effects:** Modifies SS:SP (stack segment:pointer); calls `NetISR()` (defined elsewhere).
- **Calls:** `NetISR()` (defined in another compilation unit, e.g., port.c or net.c)
- **Notes:** Uses macro-based inline stack switching (`GetStack`, `SetStack`) to preserve caller's stack and restore it on return. The private stack prevents corruption if the interrupt fires during a game function call.

### LaunchROTT
- **Signature:** `void LaunchROTT(void)`
- **Purpose:** Main launcher: set up ROTTCOM, hook the ISR, configure game parameters, and spawn the ROTT executable with appropriate arguments.
- **Inputs:** None (reads globals: `rottcom`, `pause`; reads command-line via `_argv`, `_argc`)
- **Outputs/Return:** None (process termination via `spawnv()` on success; early return if user aborts)
- **Side effects:** Calls `SetupROTTCOM()`, `getvect()`, `setvect()` (hooks interrupt); spawns child process via `spawnv()` (blocking); calls `ShutdownROTTCOM()`.
- **Calls:** `SetupROTTCOM()`, `getvect()`, `setvect()`, `sprintf()`, `printf()`, `getch()`, `spawnv()`, `ShutdownROTTCOM()`
- **Notes:** 
  - Passes raw flat address of `rottcom` to ROTT via string argument (DOS real-mode convention).
  - Builds argument array dynamically; adds "-net" marker and optionally "IS8250" serial flag if compiled with `ROTTSER`.
  - Sets `ticstep` based on `gametype`: 2 for modem (0), 1 for network (non-zero).
  - If `pause` is true, displays arguments and waits for key; ESC aborts without launching.

## Control Flow Notes
- **Initialization:** Called at launcher startup; allocates resources and hooks interrupt vector.
- **Game Execution:** ROTT executable runs as child process under `spawnv()`, with the ROTTNET_ISR installed to handle incoming network traffic asynchronously.
- **Shutdown:** On ROTT exit, `ShutdownROTTCOM()` restores the original interrupt vector and frees the stack.
- This is a DOS real-mode multiplayer launcher pattern typical of early 1990s games (IPX/modem support).

## External Dependencies
- **Standard C:** `<stdio.h>`, `<stdlib.h>`, `<string.h>` (printf, malloc, free, sprintf, sscanf)
- **DOS/Real-mode:** `<process.h>` (spawnv), `<dos.h>` (getvect, setvect, disable/enable, FP_SEG/FP_OFF), `<conio.h>` (getch)
- **Local headers:** 
  - `"rottnet.h"` (defines `rottcom_t`, constants like `ROTTLAUNCHER`)
  - `"global.h"` (Error, CheckParm, boolean, ESC constant)
  - `"port.h"` (conditional on `ROTTSER`; defines `Is8250()`, serial queue structures)
- **Defined elsewhere:** `NetISR()` (called from ROTTNET_ISR; likely in port.c or net.c), `_argc`, `_argv`, `_DS`, `_SS`, `_SP` (compiler/DOS intrinsics)
