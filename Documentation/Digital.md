# Digital I/O Components

All digital I/O components extend `Component` and implement `I_DigitalInput` or `I_DigitalOutput`. They support hardware forcing via `E_ForceState`.

---

## DigitalInput_NO (Normally Open)

Wraps a hardware `BOOL` input. `IsActive` reflects the raw input unless forced.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, Input)` | Constructor | `Input` is a `REFERENCE TO BOOL` pointing to the hardware variable |
| `IsActive` | `BOOL` (Get) | TRUE when the physical input is TRUE (or forced on) |
| `Force` | `E_ForceState` (Get/Set) | `Unforced` / `ForceOn` / `ForceOff` |

### Example

```pascal
// From DigitalInput_NO_TEST
VAR
    Input        : BOOL;
    Di           : DigitalInput_NO('PartPresent', Input);
END_VAR

// Normal operation
Di.Force := E_ForceState.Unforced;
Input    := TRUE;
// Di.IsActive -> TRUE

Input := FALSE;
// Di.IsActive -> FALSE

// Force on regardless of hardware
Di.Force := E_ForceState.ForceOn;
Input    := FALSE;
// Di.IsActive -> TRUE

// Force off regardless of hardware
Di.Force := E_ForceState.ForceOff;
Input    := TRUE;
// Di.IsActive -> FALSE
```

---

## DigitalInput_NC (Normally Closed)

Same as `DigitalInput_NO` but with inverted logic — `IsActive` is TRUE when the hardware input is FALSE.

### Interface

Identical to `DigitalInput_NO`.

| Member | Description |
|--------|-------------|
| `IsActive` | TRUE when physical input is FALSE (i.e., circuit is closed / contact not broken) |

### Example

```pascal
VAR
    Input : BOOL;
    Di    : DigitalInput_NC('EStop', Input);
END_VAR

Input := FALSE;   // circuit closed — no E-Stop
// Di.IsActive -> TRUE

Input := TRUE;    // circuit open — E-Stop triggered
// Di.IsActive -> FALSE
```

---

## DigitalOutput

Wraps a hardware `BOOL` output. Writing to `On` drives the hardware variable.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, Output)` | Constructor | `Output` is a `REFERENCE TO BOOL` pointing to the hardware variable |
| `On` | `BOOL` (Get/Set) | Command the output; drives the hardware variable |
| `Force` | `E_ForceState` (Get/Set) | `Unforced` / `ForceOn` / `ForceOff` |

### Example

```pascal
VAR
    DoSeal  : BOOL;
    Seal    : DigitalOutput('SealSolenoid', DoSeal);
END_VAR

Seal.On := TRUE;   // DoSeal -> TRUE (hardware coil energised)
Seal.On := FALSE;  // DoSeal -> FALSE

// Force on regardless of command
Seal.Force := E_ForceState.ForceOn;
Seal.On    := FALSE;
// DoSeal -> TRUE (forced)
```

---

## DigitalInput_Combiner

Combines two `I_DigitalInput` references with AND or OR logic into a single `IsActive` signal.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, InputA, InputB, Mode)` | Constructor | `Mode`: `E_CombinerMode.And` or `E_CombinerMode.Or` |
| `IsActive` | `BOOL` (Get) | Result of the logical combination |

### Example

```pascal
VAR
    DiA, DiB   : BOOL;
    InputA     : DigitalInput_NO('SensorA', DiA);
    InputB     : DigitalInput_NO('SensorB', DiB);
    Combiner   : DigitalInput_Combiner('BothPresent', InputA, InputB, E_CombinerMode.And);
END_VAR

DiA := TRUE; DiB := TRUE;
// Combiner.IsActive -> TRUE

DiA := TRUE; DiB := FALSE;
// Combiner.IsActive -> FALSE
```

---

## DigitalInput_DebounceRisingEdge / DigitalInput_DebounceFallingEdge

Wraps a digital input and only passes the state change after it has been stable for a configurable debounce time.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, Input, DebounceTime)` | Constructor | `DebounceTime`: `TIME` value for stability window |
| `IsActive` | `BOOL` (Get) | Debounced output |
| `CyclicLogic()` | Method | Must be called each scan |
