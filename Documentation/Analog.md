# Analog I/O Components

Analog components wrap hardware `INT` variables and apply a configurable scaling and calibration pipeline to produce engineering-unit `LREAL` values.

---

## AnalogInput

Reads a raw `INT` from hardware, scales it linearly, then applies calibration to produce an `LREAL` value in engineering units.

**Processing chain (read):** `RawValue → Scale.Transform() → Calibrate.Transform() → Value`

Extends: `Component`  
Implements: `I_AnalogInput`, `I_AnalogScalable`, `I_CalibratableLinear`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, Input)` | Constructor | `Input` is a `REFERENCE TO INT` pointing to the hardware variable |
| `RawValue` | `INT` (Get) | Forced or actual hardware integer |
| `Value` | `LREAL` (Get) | Scaled and calibrated engineering-unit value |
| `Scale` | `I_AnalogScale` (Get/Set) | Linear scaling object; defaults to `AnalogScale` |
| `CalibrateLinear` | `I_CalibrateLinear` (Get/Set) | Linear calibration applied after scaling |
| `CalibrateCustom` | `I_TransformLRealValue` (Set) | Custom calibration transform (replaces linear) |
| `Force` | `E_ForceState` (Get/Set) | `Unforced` / `ForceOn` / `ForceOff` |
| `ForceValue` | `INT` (Get/Set) | Raw integer used when forced on |

### Example

```pascal
VAR
    RawHw     : INT;
    Pressure  : AnalogInput('InletPressure', RawHw);
    Scale     : AnalogScale;
END_VAR

// Configure 0–32767 raw -> 0.0–10.0 bar
Scale.InputMinimum  := 0;
Scale.InputMaximum  := 32767;
Scale.OutputMinimum := 0.0;
Scale.OutputMaximum := 10.0;
Pressure.Scale      := Scale;

RawHw := 16383;           // mid-scale
// Pressure.Value -> ~5.0 bar
```

---

## AnalogOutput

Accepts an engineering-unit `LREAL` command and writes the inverse-scaled `INT` to the hardware output variable.

**Processing chain (write):** `Value → Calibrate.Transform(Reverse) → Scale.Transform(Reverse) → RawValue → Output`

Extends: `Component`  
Implements: `I_AnalogOutput`, `I_AnalogScalable`, `I_CalibratableLinear`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, Output)` | Constructor | `Output` is a `REFERENCE TO INT` pointing to the hardware variable |
| `RawValue` | `INT` (Get/Set) | Direct raw integer written to hardware |
| `Value` | `LREAL` (Get/Set) | Engineering-unit command; setting this triggers the inverse transform |
| `Scale` | `I_AnalogScale` (Get/Set) | Same scale object as `AnalogInput` |
| `Force` | `E_ForceState` (Get/Set) | Forcing support |

### Example

```pascal
VAR
    RawHw  : INT;
    Speed  : AnalogOutput('ConveyorSpeed', RawHw);
    Scale  : AnalogScale;
END_VAR

Scale.InputMinimum  := 0;
Scale.InputMaximum  := 32767;
Scale.OutputMinimum := 0.0;
Scale.OutputMaximum := 100.0;   // percent
Speed.Scale         := Scale;

Speed.Value := 50.0;            // 50 %
// RawHw -> 16383 (inverse transform)
```

---

## AnalogScale

Linear scaling utility shared by both analog input and output.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `InputMinimum` / `InputMaximum` | `LREAL` (Get/Set) | Raw value range |
| `OutputMinimum` / `OutputMaximum` | `LREAL` (Get/Set) | Engineering-unit range |
| `Transform(Value, Reverse)` | Method | Scales forward (`Reverse = FALSE`) or backward (`Reverse = TRUE`) |

Implements: `I_AnalogScale`
