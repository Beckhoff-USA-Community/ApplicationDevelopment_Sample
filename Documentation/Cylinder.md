# Cylinder

Double-acting pneumatic or hydraulic cylinder controller. Manages a two-output (extend/retract solenoid) / two-input (extended/retracted sensor) mechanism through a built-in state machine with timeout and plausibility error detection.

Extends: `CyclicComponent`  
Implements: `I_Cylinder`

## Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, ExtendOutput, RetractOutput, ExtendedInput, RetractedInput)` | Constructor | All I/O passed as interfaces; null devices are created automatically if omitted |
| `ExtendTimeout` | `TIME` (Get/Set) | Max time allowed to reach extended position (default configurable) |
| `RetractTimeout` | `TIME` (Get/Set) | Max time allowed to reach retracted position |
| `Extend()` | Method | Commands the cylinder to extend |
| `Retract()` | Method | Commands the cylinder to retract |
| `Reset()` | Method | Clears error state |
| `CyclicLogic()` | Method | Must be called each scan |
| `Extended` | `BOOL` (Get) | TRUE when fully extended sensor is active |
| `Extending` | `BOOL` (Get) | TRUE while moving toward extended position |
| `Retracted` | `BOOL` (Get) | TRUE when fully retracted sensor is active |
| `Retracting` | `BOOL` (Get) | TRUE while moving toward retracted position |
| `Busy` | `BOOL` (Get) | TRUE while extending or retracting |
| `Error` | `BOOL` (Get) | TRUE when a fault is active |
| `ErrorId` | `E_CylinderErrorId` (Get) | Active error code |

### Error Codes (`E_CylinderErrorId`)

| Code | Cause |
|------|-------|
| `ExtendTimeout` | Extended sensor not reached within `ExtendTimeout` |
| `RetractTimeout` | Retracted sensor not reached within `RetractTimeout` |
| `ExtendedPlausibility` | Extended sensor lost while not retracting |
| `RetractedPlausibility` | Retracted sensor lost while not extending |
| `BothSensorsActive` | Both sensors active simultaneously |

## Example

```pascal
// From Cylinder_TEST — declaration
VAR
    DiExtended : BOOL;
    DiRetracted : BOOL;
    DoExtend   : BOOL;
    DoRetract  : BOOL;

    Extend   : DigitalOutput('Extend', DoExtend);
    Retract  : DigitalOutput('Retract', DoRetract);
    Extended : DigitalInput_NO('Extended', DiExtended);
    Retracted : DigitalInput_NO('Retracted', DiRetracted);

    Cyl : Cylinder('SealCylinder', Extend, Retract, Extended, Retracted)
        := (ExtendTimeout := T#250MS, RetractTimeout := T#250MS);
END_VAR

// Extend command
Cyl.Extend();
Cyl.CyclicLogic();
// Cyl.Extending -> TRUE

// Simulate sensor arriving
DiExtended := TRUE;
Cyl.CyclicLogic();
// Cyl.Extended -> TRUE

// Retract
DiExtended  := FALSE;
DiRetracted := TRUE;
Cyl.Retract();
Cyl.CyclicLogic();
// Cyl.Retracted -> TRUE

// Both sensors simultaneously — fault
DiExtended  := TRUE;
DiRetracted := TRUE;
Cyl.CyclicLogic();
// Cyl.Error   -> TRUE
// Cyl.ErrorId -> E_CylinderErrorId.BothSensorsActive

Cyl.Reset();
Cyl.CyclicLogic();
// Cyl.Error -> FALSE
```

## Notes

- `Cylinder_TcEvents` wraps a `Cylinder` and publishes its errors as TwinCAT events. Instantiate alongside the cylinder and call `CyclicLogic()` each scan.
- If no I/O interfaces are supplied, the constructor creates null device stubs so the FB compiles and runs without hardware connected.
