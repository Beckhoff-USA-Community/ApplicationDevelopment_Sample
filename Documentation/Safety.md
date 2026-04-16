# Safety Components

## SafetyBase

Abstract base for TwinCAT Safety function blocks. Reads `ST_SafetyStateDiag` from the TwinSAFE PLC via hardware mapping, drives `OK`/`Error`/`Busy` status, and supports simulation mode for commissioning without safety hardware.

Extends: `CyclicComponent`  
Implements: `I_SafetyBase`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name)` | Constructor | Standard component name |
| `OK` | `BOOL` (Get) | TRUE when the safety function is in a safe, running state |
| `Error` | `BOOL` (Get) | TRUE on a safety fault |
| `ErrorId` | `UDINT` (Get) | Error code |
| `Busy` | `BOOL` (Get) | TRUE while an operation is in progress |
| `Simulated` | `BOOL` (Get/Set) | Bypasses hardware check; useful during commissioning without safety hardware |
| `ResetFunction` | `I_SafetyReset` (Set) | Optional reset handler called by `Reset()` |
| `Reset()` | Method | Executes the registered reset function |
| `SimulateSafetyState(Value)` | Method | Injects a simulated state value into `InfoData.State` |
| `SimulateSafetyDiag(Value)` | Method | Injects a simulated diagnostic value into `InfoData.Diag` |
| `CyclicLogic()` | Method | Checks hardware mapping; drives reset function each scan |

### Notes

- `InfoData` must be linked `AT %I*` to the TwinSAFE PLC's state/diagnostic output in the hardware configuration.
- If not mapped and `Simulated = FALSE`, a trace warning is logged once and the FB runs without diagnostics.

---

## SafetyAndOrFB

Extends `SafetyBase`. Sets `OK = TRUE` when `InfoData.State = E_SafetyOrAnd_State.Run`. Use this for TwinSAFE AND/OR gate function blocks.

Extends: `SafetyBase`

### Example

```pascal
VAR
    EStopGroup : SafetyAndOrFB('EStopGroup');
END_VAR

EStopGroup.CyclicLogic();

IF EStopGroup.OK THEN
    // All E-Stop inputs are clear, safe to operate
END_IF

// Simulation (no hardware needed)
EStopGroup.Simulated := TRUE;
EStopGroup.SimulateSafetyState(E_SafetyOrAnd_State.Run);
```

---

## SafetyModule

A specialised `Module` that acts as a container for all safety-related `SafetyBase` components. Manages the TwinSAFE PLC connection via CoE, reads FSoE address and project CRC at startup, provides aggregate OK/Error status across all registered safety components, and implements a timed reset sequence for TwinSAFE.

Extends: `Module`  
Implements: `I_SafetyModule`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name)` | Constructor | Registers internal `CoeDevice` with the module |
| `OK` | `BOOL` (Get) | TRUE when all registered safety components report OK |
| `Error` | `BOOL` (Get) | TRUE if any registered component has an error |
| `ErrorId` | `UDINT` (Get) | Error ID from the first faulted component |
| `Busy` | `BOOL` (Get) | TRUE while a reset sequence is running |
| `Simulated` | `BOOL` (Get/Set) | Propagates simulation mode to all child components via `SetSafetySimulationVisitor` |
| `AddSafetyComponent(Component)` | Method | Registers an `I_SafetyBase` component with the module |
| `Reset()` | Method | Triggers a timed TwinSAFE reset sequence (250 ms delay) |
| `SafetyAddressInfo` | `REFERENCE TO ST_SafetyAddressInfo` (Get) | FSoE address, serial number, and CRC read during init |
| `SafetyPlcCoE` | `I_CoeDevice` (Get) | CoE device used to communicate with the TwinSAFE PLC |

### Hardware Requirements

The following variables must be linked `AT %I*` in the hardware configuration:
- `SafetyPLC_WcState` — Working counter state of the safety PLC frame
- `SafetyPLC_AmsAddr` — AMS address of the TwinSAFE PLC

### Example

```pascal
VAR
    Safety     : SafetyModule('Safety');
    EStop      : SafetyAndOrFB('EStop');
    LightCurtain : SafetyAndOrFB('LightCurtain');
END_VAR

Safety.AddSafetyComponent(EStop);
Safety.AddSafetyComponent(LightCurtain);

// Each scan
Safety.CyclicLogic();
EStop.CyclicLogic();
LightCurtain.CyclicLogic();

IF NOT Safety.OK THEN
    // A safety function is not in Run state
END_IF

// Reset after a safety event is cleared
Safety.Reset();
```

---

## SafetyResetPulse

Generates a pulsed reset signal for TwinSAFE function blocks. Implements `I_SafetyReset` and is typically injected into a `SafetyBase` via `ResetFunction`.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `Execute()` | Method | Triggers the reset pulse |
| `CyclicLogic()` | Method | Must be called each scan to manage the pulse timing |
