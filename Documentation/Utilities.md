# Utilities

---

## Trace

Global utility for debug logging and system diagnostics. Broadcasts messages to TcEventLogger and AdsLogger, both rate-limited to `MaxTraceMessagesPerScan`. Logging to either sink can be toggled via `EnableTcEventLogging` and `EnableAdsLogger` in the global parameters.

```pascal
Trace.LogMessage(Key := 'SourceName', Value := 'Message text');
```

`Key` is the source label; `Value` is the message body. Output format: `Value: Key`.

**Register a custom transport** (file logging, MQTT, etc.):

```pascal
Trace.Subscribe(myCustomLogger); // myCustomLogger implements I_TraceLogger
```

---

## ForcibleBool

A `BOOL` wrapper that supports three states: normal operation, forced on, or forced off. Used inside digital I/O components to implement hardware overrides.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `Input` | `BOOL` (Set) | The normal requested state |
| `Output` | `BOOL` (Get) | The resolved state after applying force logic |
| `CurrentForceState` | `E_ForceState` (Get) | Current force mode |
| `SetForceState(ForceState)` | Method | Changes force mode and raises/clears TcEvent |
| `ResolveOutput()` | Method | Recomputes output from input and force state |

**Logic:**
- `Unforced` → `Output = Input`
- `ForceOn` → `Output = TRUE` (raises alarm event)
- `ForceOff` → `Output = FALSE` (raises alarm event)

---

## ForcibleInt

Same pattern as `ForcibleBool` but for `INT` values. Used inside `AnalogInput` / `AnalogOutput`.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `Input` | `INT` (Set) | Normal requested value |
| `Output` | `INT` (Get) | Resolved value after force |
| `ForceValue` | `INT` (Get/Set) | Value used when forced on |
| `CurrentForceState` | `E_ForceState` (Get) | Current force mode |
| `SetForceState(ForceState)` | Method | Changes force mode |

---

## AnalogScale

Linear mapping between a raw integer range and an engineering-unit range. Used by `AnalogInput` and `AnalogOutput` for forward and reverse scaling.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `InputMinimum` / `InputMaximum` | `LREAL` (Get/Set) | Raw value bounds |
| `OutputMinimum` / `OutputMaximum` | `LREAL` (Get/Set) | Engineering-unit bounds |
| `Transform(Value, Reverse)` | Method → `LREAL` | `Reverse = FALSE` scales raw → EU; `Reverse = TRUE` scales EU → raw |

Implements: `I_AnalogScale`

### Example

```pascal
VAR
    Scale : AnalogScale;
END_VAR

Scale.InputMinimum  := 0;
Scale.InputMaximum  := 32767;
Scale.OutputMinimum := 0.0;
Scale.OutputMaximum := 100.0;

Result := Scale.Transform(16383.0, FALSE);
// -> ~50.0 (forward: raw to engineering units)

Result := Scale.Transform(50.0, TRUE);
// -> ~16383.0 (reverse: engineering units to raw)
```

---

## Stopwatch

DC-time-based stopwatch with nanosecond precision. Useful for performance measurement and timing diagnostics.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `Start()` | Method | Begins timing |
| `Stop()` | Method → `LREAL` | Ends timing; returns elapsed seconds |
| `Lap()` | Method → `LREAL` | Returns current elapsed without stopping |
| `Elapsed` | `LREAL` (Get) | Elapsed time in seconds |
| `Running` | `BOOL` (Get) | TRUE while timing |

### Example

```pascal
VAR
    SW : Stopwatch;
END_VAR

SW.Start();
// ... do work ...
ElapsedSec := SW.Stop();
```

---

## RecipeManagement

Compares an "update" recipe buffer against the active recipe and optionally auto-applies changes.

Extends: `CyclicComponent`  
Implements: `I_RecipeManagement`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `ChangesPending` | `BOOL` (Get) | TRUE when update buffer differs from active |
| `AutoLoadChangesToActive` | `BOOL` (Get/Set) | If TRUE, changes are applied automatically each scan |
| `LoadChangesToActive()` | Method | Manually copies update buffer to active |
| `CompareData()` | Method | Triggers a MEMCMP; updates `ChangesPending` |
| `CyclicLogic()` | Method | Must be called each scan |
