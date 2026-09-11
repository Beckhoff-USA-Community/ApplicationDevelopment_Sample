# Motion

Axis components for TwinCAT motion control. The framework ships **two parallel, feature-equivalent axis stacks** — one built on the classic **MC2** libraries (`Tc2_MC2`) and one built on the **MC3** libraries (`Tc3_Mc3Base` / `Tc3_Mc3Ptp`) — behind a common, technology-neutral set of interfaces.

Every axis is a `CyclicComponent`, so it registers with its parent module, participates in the initializer, and is visited by the same visitors as any other component.

> **New in V2.0.0** — the MC3 stack was added and all pre-existing motion types were renamed with an `Mc2` prefix. See [Migrating from V1.x](#migrating-from-v1x) for the full rename table.
>
> **New in V2.1.0** — the shared abstraction no longer exposes `AXIS_REF` or any motion-library enum, so the
> HMI and event-reaction function blocks are now generation-agnostic and shared. See
> [Migrating from V2.0.x](#migrating-from-v20x).

## Folder Layout

| Folder | Contents |
|--------|----------|
| `Motion/` | `AxisPTP_HMI`, `Axis_TcEvents` — generation-agnostic, shared by MC2 and MC3 |
| `Motion/Interface/` | Technology-neutral axis interfaces (`I_Axis`, `I_Axis_PTP`, `I_AxisStatus`, `I_AxisJog`, …) — shared by MC2 and MC3 |
| `Motion/DUT/` | `ST_AxisPTP_HMI` — shared HMI symbol struct; `E_AxisJogMode` — generation-neutral jog mode |
| `Motion/MC2/` | `Mc2Axis`, `Mc2AxisPTP`, `Mc2SlaveAxisPTP` + `I_Mc2*` |
| `Motion/MC3/` | `Mc3Axis`, `Mc3AxisPTP`, `Mc3SlaveAxisPTP` + `I_Mc3*` |
| `_Internal/MotionTasks/MC2/` | MC2 motion tasks (move, power, home, reset, coupling) |
| `_Internal/MotionTasks/MC3/` | MC3 motion tasks (move, power, home, reset, gearing) |
| `Utilities/Motion/` | Standalone motion maths helpers (see [Motion Helpers](#motion-helpers)) |

## Choosing MC2 or MC3

| Use | When |
|-----|------|
| `Mc3AxisPTP` | New projects. MC3 is the current Beckhoff motion API — object-oriented axis references, enum-based settings, cleaner error model |
| `Mc2AxisPTP` | Existing MC2 applications, or when an MC2-only feature is needed — multi-master gearing, touch-probe homing, SoE reset, `ST_MoveOptions`, or the full `ST_AxisParameterSet` |

Both stacks expose the same method and property names, so application code written against `I_Axis_PTP` — or the finer-grained `I_AxisStatus`, `I_AxisJog`, `I_AxisMoveAbsolute`, … — is portable between them. The raw axis reference (`Tc2_MC2.AXIS_REF` vs `Tc3_Mc3Ptp.AXIS_REF`) and the generation-specific settings enums are reachable only through `I_Mc2Axis` / `I_Mc3Axis`, never through the shared abstraction.

## Shared Axis Interfaces

These live in `Motion/Interface/` and are implemented by both `Mc2Axis` and `Mc3Axis`.

| Interface | Extends | Members |
|-----------|---------|---------|
| `I_Axis` | `I_Component`, `I_AxisEnable`, `I_Resettable`, `I_Stoppable`, `I_AxisDynamics`, `I_AxisStatus`, `I_AxisJog`, `I_AxisHome`, `I_AxisSettings`, `I_Override` | `ErrorId`, `MotionTasks` — the full generation-neutral axis |
| `I_Axis_PTP` | `I_Axis`, `I_AxisMove*` | The generation-neutral point-to-point axis |
| `I_AxisStatus` | `I_Base`, `I_TaskResult` | `ActualPosition`, `ActualPositionModulo`, `ActualVelocity`, `ActualAcceleration`, `ActualTorque`, `SetPosition`, `SetPositionModulo`, `SetVelocity`, `SetAcceleration`, `PositionLag`, `Enabled`, `Disabled`, `Coupled`, `SynchronizedMotion`, `Stopped` |
| `I_AxisSettings` | `I_Base` | `JogMode` (`E_AxisJogMode`) |
| `I_MotionTaskCollection` | `I_TaskResult` | `ErrorTask` — read-only view on the axis's task collection |
| `I_AxisDynamics` | `I_Base` | `Velocity`, `Acceleration`, `Deceleration`, `Jerk` |
| `I_AxisEnable` | `I_Enablable` | `InhibitFeedForward`, `InhibitFeedBackward` |
| `I_AxisHome` | `I_Base` | `Home()` |
| `I_AxisJog` | `I_Base` | `Jog(JogForward, JogBackwards, Distance := 0)` |
| `I_AxisMoveAbsolute` | `I_Base` | `MoveAbsolute(TargetPosition, AbortPrevious := TRUE)` |
| `I_AxisMoveRelative` | `I_Base` | `MoveRelative(Distance, AbortPrevious := TRUE)` |
| `I_AxisMoveVelocity` | `I_Base` | `MoveVelocity(Velocity, AbortPrevious := TRUE)` |
| `I_AxisMoveModulo` | `I_Base` | `MoveModulo(Position, AbortPrevious := TRUE)` |

`I_TaskResult` contributes `Busy`, `Error` and `ErrorId` — the same result contract used by the motion tasks themselves.

The technology-specific interfaces compose these:

```
I_Mc3Axis      EXTENDS I_Axis, I_Mc3Settings
I_Mc3Axis_PTP  EXTENDS I_Mc3Axis, I_Axis_PTP
I_Mc3SlaveAxis EXTENDS I_Mc3Axis_PTP, I_Mc3AxisGear
```

`I_Mc2Axis`, `I_Mc2Axis_PTP` and `I_Mc2SlaveAxis` are structured identically.

Note where the raw `AXIS_REF` sits. `I_Mc3Axis` extends `I_Mc3AxisRef` (`I_Mc2AxisRef` for MC2), so the full
NC structure is directly available for raw access:

```pascal
VAR
    MyAxis : I_Mc3Axis_PTP;
END_VAR

AxisId     := MyAxis.Axis.McToPlc.Std.AxisId;
DriveError := MyAxis.Axis.McToPlc.Std.HasError;
```

`I_Axis` and `I_Axis_PTP` deliberately do **not** carry it, and that is the whole point of the split: the
shared abstraction stays free of `Tc2_MC2` / `Tc3_Mc3Ptp` types so one HMI and one event reaction can serve
both stacks, while `I_Mc2Axis` / `I_Mc3Axis` remain the escape hatch for generation-specific work. Typing a
variable as `I_Mc3Axis` is a deliberate statement that the code is MC3-only — it already implies the MC3
settings enums from `I_Mc3Settings`, so the axis reference adds no new coupling.

## Function Blocks

Both stacks follow an identical three-level inheritance chain.

| MC2 | MC3 | Extends | Adds |
|-----|-----|---------|------|
| `Mc2Axis` | `Mc3Axis` | `CyclicComponent` | Enable/disable, stop, reset, home, jog, dynamics, status, override |
| `Mc2AxisPTP` | `Mc3AxisPTP` | `…Axis` | `MoveAbsolute`, `MoveRelative`, `MoveVelocity`, `MoveModulo` |
| `Mc2SlaveAxisPTP` | `Mc3SlaveAxisPTP` | `…AxisPTP` | Electronic gearing to a master axis |
| `AxisPTP_HMI` (shared) | `AxisPTP_HMI` (shared) | `HmiFunction` | Operator faceplate with permissives |
| `Axis_TcEvents` (shared) | `Axis_TcEvents` (shared) | `CyclicComponent` | Raises a TwinCAT event when a motion task faults |

### `Mc3Axis` / `Mc2Axis`

Extends: `CyclicComponent`
Implements: `I_Initializable`, `I_Mc3Axis` / `I_Mc2Axis`

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, Axis)` | Constructor | `Axis` is a `REFERENCE TO Tc3_Mc3Ptp.AXIS_REF` (MC3) or `Tc2_MC2.AXIS_REF` (MC2) |
| `Initialize()` | Method | Reads `DefaultAcceleration`, `DefaultDeceleration`, `DefaultJerk` and `MaximumVelocity` from the NC axis and pre-loads the dynamics properties. Driven automatically from `CyclicLogic()` until complete |
| `CyclicLogic()` | Method | Initializes on the first scans, then runs the motion task collection. Must be called each scan |
| `Enable()` / `Disable()` | Method | Power on / off via the internal power task. `Enable()` is ignored while `Error` is set |
| `Stop()` | Method | Runs the stop task. Refused while the NC axis itself is in `ErrorStop` — reset first |
| `Reset()` | Method | Runs the reset task, then clears the task collection |
| `Home()` | Method | Runs the home task |
| `Jog(JogForward, JogBackwards, Distance)` | Method | Runs the jog task using the configured `JogMode` |
| `Velocity`, `Acceleration`, `Deceleration`, `Jerk` | `LREAL` (Get/Set) | Dynamics applied to every subsequent move |
| `Override` | `LREAL` (Get/Set) | Velocity override in percent (0–100) |
| `Busy`, `Error`, `ErrorId` | Get | Aggregated from the motion task collection; `Error` also covers the NC axis being in `ErrorStop` |
| `MotionTasks` | `I_MotionTaskCollection` (Get) | Read-only view on the axis's task collection — `Busy`, `Error`, `ErrorId`, `ErrorTask` |
| `StopTask`, `ResetTask`, `HomeTask` | `I_Mc3MotionTask` (Get/Set) | Swap a default task for a project-specific one; the collection is re-registered automatically |
| `Coupled`, `SynchronizedMotion`, `Disabled` | `BOOL` (Get) | NC status, mapped from each generation's own status model |
| `JogMode` | `E_AxisJogMode` (Get/Set) | `Slow` or `Fast`; mapped onto the generation's own jog enum |
| `Axis` | `REFERENCE TO AXIS_REF` (Get) | The underlying NC axis reference, for raw access to the NC structure. On `I_Mc3Axis` / `I_Mc2Axis`, not on the shared `I_Axis` |
| `InhibitFeedForward` / `InhibitFeedBackward` | `BOOL` (Get/Set) | Travel-direction inhibits |
| `Initialized` | `BOOL` (Get) | TRUE once the parameter read has completed |

**Generation-specific settings** (`I_Mc3Settings` / `I_Mc2Settings`) are exposed as references, so they can be
written in place. Reaching them means typing the variable as `I_Mc3Axis` / `I_Mc2Axis` rather than `I_Axis`:

| Property | MC3 type | MC2 type |
|----------|----------|----------|
| `BufferMode` | `Tc3_Mc3Ptp.EBufferMode` (default `Aborting`) | `Tc2_MC2.MC_BufferMode` |
| `Direction` | `Tc3_Mc3Ptp.EModuloDirection` (default `ShortestDistance`) | `Tc2_MC2.MC_Direction` (default `MC_Shortest_Way`) |
| `MoveOptions` | — | `Tc2_MC2.ST_MoveOptions` |
| `AxisParameterSet` | — | `Tc2_MC2.ST_AxisParameterSet`, read during `Initialize()` |

`JogMode` is **not** here — it moved to the shared `I_AxisSettings` as a generation-neutral `E_AxisJogMode`
(`Slow` / `Fast`), passed by value, and each axis maps it onto its own library enum.

### `Mc3SlaveAxisPTP` / `Mc2SlaveAxisPTP`

A PTP axis that can be electronically geared to a master. The coupling is itself a motion task, so it reports `Busy` / `Error` through the same collection as every other move.

**MC3 (`I_Mc3AxisGear`):**

| Member | Type | Description |
|--------|------|-------------|
| `Master` | `I_Mc3Axis` (Get/Set) | The master axis |
| `RatioNumerator`, `RatioDenominator` | `LREAL` (Get/Set) | Gear ratio |
| `GearIn()` | Method | Couples to the master using the current `BufferMode` |
| `InGear` | `BOOL` (Get) | TRUE while coupled |
| `MasterHasError` | `BOOL` (Get) | TRUE when the master axis is faulted |
| `ReactionToMasterError`, `ReactionToSlaveError` | Enum (Get/Set) | Coupling fault behaviour |

**MC2 (`I_Mc2AxisGear`)** supports up to four masters instead: `Master1`–`Master4`, `GearRatioMaster1`–`GearRatioMaster4`, `GearInMultiMasterOptions`, plus explicit `GearIn()` / `GearOut()`.

### `AxisPTP_HMI`

Extends: `HmiFunction`

One function block for both stacks — it depends only on `I_Axis_PTP` and never touches a motion-library type.

`FB_Init(Name, Axis, Unit)` — `Axis` is an `I_Axis_PTP` (so any `Mc2AxisPTP` or `Mc3AxisPTP`), `Unit` is the display unit (e.g. `'°'`, `'mm'`).

Publishes an `ST_AxisPTP_HMI` symbol carrying live status (`ActualPosition`, `ActualVelocity`, `ActualAcceleration`, `ActualTorque`, `ActualPositionModulo`, `Busy`, `Error`, `ErrorId`) and one `Button` per command: `Enable`, `Disable`, `Stop`, `Reset`, `Home`, `JogForwardSlow`, `JogBackwardSlow`, `JogForwardFast`, `JogBackwardFast`, `MoveAbsolute`, `MoveRelative`, `MoveVelocity`, `MoveModulo` — with `TargetPosition`, `TargetDistance` and `TargetVelocity` as setpoints.

Every command has a matching `…Permissive` property (`EnablePermissive`, `MoveAbsolutePermissive`, …) so the application can gate operator access. Call `CyclicLogic()` each scan. See [HMI](HMI.md).

### `Axis_TcEvents`

Extends: `CyclicComponent`
Implements: `I_EventReactionTcEvents`

One function block for both stacks — it depends only on `I_Axis`.

`FB_Init(Name, Axis)` — `Axis` is an `I_Axis`. Each scan it checks `Axis.MotionTasks.Error` and raises `E_McAxis.AxisError` with the failing task name and error id as arguments. `SetEventReactions()` maps severities as with any other `TcEventClass` consumer. See [Events](Events.md).

## Motion Tasks

Every axis command is implemented as a **motion task** — a small function block wrapping one `Tc2_MC2` / `Tc3_Mc3Ptp` function block. Tasks are registered in a generic `Mc3MotionTaskCollection<MaxMotionTasks>` (or `Mc2…`) owned by the axis, which calls each task cyclically and aggregates their results.

| Member | Description |
|--------|-------------|
| `AddTask(Task)` / `RemoveTaskByInstance(Task)` | Register / unregister a task |
| `CyclicLogic()` | Runs every registered task; called from the axis |
| `Busy`, `Error`, `ErrorId` | Aggregated result across all tasks |
| `ErrorTask` | Name of the task that faulted |
| `Reset()` | Clears the aggregated error state |

Tasks derive from the abstract `Mc3MotionTask` / `Mc2MotionTask`, which implements `I_Mc3MotionTask` (`EXTENDS I_Cyclic, I_TaskResult, I_Command`). A project can therefore add its own task with `MotionTasks.AddTask()`, or substitute one of the defaults via the `StopTask` / `ResetTask` / `HomeTask` properties.

Defaults come from `ST_Mc3MotionDefaults` / `ST_Mc2MotionDefaults`: CoE reset, set-zero-here homing, and stop.

### Task Catalogue

| Task | MC2 | MC3 |
|------|-----|-----|
| Power on/off | `Mc2PowerTask` | `Mc3PowerTask` |
| Stop | `Mc2StopTask` | `Mc3StopTask` |
| Jog | `Mc2MoveJogTask` | `Mc3MoveJogTask` |
| Move absolute | `Mc2MoveAbsoluteTask` | `Mc3MoveAbsoluteTask` |
| Move relative | `Mc2MoveRelativeTask` | `Mc3MoveRelativeTask` |
| Move velocity | `Mc2MoveVelocityTask` | `Mc3MoveVelocityTask` |
| Move modulo | `Mc2MoveModuloTask` | `Mc3MoveModuloTask` |
| Gearing | `Mc2MotionCoupleTask_Gearing` | `Mc3MotionCoupleTask_Gearing` |
| Home — set zero here | `Mc2MotionHomeTask_SetZeroHere` | `Mc3MotionHomeTask_SetZeroHere` |
| Home — set position | `Mc2MotionHomeTask_SetPosition` | `Mc3MotionHomeTask_SetPosition` |
| Home — set position offset | `Mc2MotionHomeTask_SetPositionOffset` | — |
| Home — CoE encoder offset | `Mc2MotionHomeTask_CoESetEncoderOffset` | — |
| Home — touch probe | `Mc2MotionHomeTask_TouchProbe` | — |
| Reset — CoE | `Mc2CoEResetTask` | `Mc3CoEResetTask` |
| Reset — CoE extended | `Mc2CoEExtendedResetTask` | — |
| Reset — SoE | `Mc2SoEResetTask` | — |
| AX5000 current-to-torque | `Mc2MotionTask_AX5000AmpToNm` | — |

## Configuration

`ApplicationBase/ApplicationBaseMotionParameter.TcGVL`:

- `MaxMotionTasks`: 10 — the generic bound on `Mc2MotionTaskCollection` / `Mc3MotionTaskCollection`. Raise it if an axis carries more than ten registered tasks.

## Example

From the VFFS demo — an MC3 master/slave pair with faceplates and events (see [VFFS](VFFS.md)):

```pascal
VAR
    PullWheelLeftAxisRef  : AXIS_REF;
    PullWheelRightAxisRef : AXIS_REF;

    _PullWheelLeftAxis  : Mc3AxisPTP('Pull Wheel Left Axis', PullWheelLeftAxisRef);
    _PullWheelRightAxis : Mc3SlaveAxisPTP('Pull Wheel Right Axis', PullWheelRightAxisRef);

    PullWheelLeftAxis_HMI     : AxisPTP_HMI('Pull Wheel Left Axis HMI', _PullWheelLeftAxis, '°');
    PullWheelLeftAxis_Events  : Axis_TcEvents('Pull Wheel Left Axis Events', _PullWheelLeftAxis);
    PullWheelRightAxis_HMI    : AxisPTP_HMI('Pull Wheel Right Axis HMI', _PullWheelRightAxis, '°');
    PullWheelRightAxis_Events : Axis_TcEvents('Pull Wheel Right Axis Events', _PullWheelRightAxis);
END_VAR
```

```pascal
// Gear the right wheel to the left one, then drive the pair
_PullWheelRightAxis.Master           := _PullWheelLeftAxis;
_PullWheelRightAxis.RatioNumerator   := 1.0;
_PullWheelRightAxis.RatioDenominator := 1.0;
_PullWheelRightAxis.GearIn();

_PullWheelLeftAxis.Velocity := Recipe.LeftAxis.Velocity;
_PullWheelLeftAxis.MoveRelative(Recipe.Length);

IF NOT _PullWheelLeftAxis.Busy AND NOT _PullWheelLeftAxis.Error THEN
    // move complete
END_IF
```

Axes are components: register them with the owning module so `Initialize()` and `CyclicLogic()` are driven for you, exactly as for any other `CyclicComponent`. See [Module](Module.md).

## Motion Helpers

Standalone maths helpers in `Utilities/Motion/`, usable independently of the axis components:

| POU | Purpose |
|-----|---------|
| `ATAN2` | Four-quadrant arctangent (`FUNCTION`) |
| `CalculateHypotenuseAngle` | Hypotenuse and angle from opposite/adjacent sides, in radians and degrees |
| `CalculateMotionDynamics` | Gentlest velocity/acceleration/deceleration/jerk set that still covers a given distance in a given time without exceeding the supplied maxima (7-phase jerk-limited profile) |
| `CamSmoothDerivatives` | Spline-smooths a cam table and reports maximum slave velocity, acceleration and jerk |
| `XPlanar6DMoverToWorldTransform` | Forward-transforms an XPlanar mover tool pose into tile world coordinates (intrinsic XYZ Tait-Bryan) |

## Migrating from V1.x

V2.0.0 renamed every motion type with an explicit `Mc2` prefix to make room for the MC3 stack. This is a **breaking change** — application code must be updated. The mapping is purely mechanical:

| V1.x | V2.0.0 |
|------|--------|
| `Axis` | `Mc2Axis` |
| `AxisPTP` | `Mc2AxisPTP` |
| `SlaveAxisPTP` | `Mc2SlaveAxisPTP` |
| `AxisPTP_HMI` | `Mc2AxisPTP_HMI` |
| `Axis_TcEvents` | `Mc2Axis_TcEvents` |
| `I_Axis` | `I_Mc2Axis` |
| `I_Axis_PTP` | `I_Mc2Axis_PTP` |
| `I_SlaveAxis` | `I_Mc2SlaveAxis` |
| `I_AxisGear` | `I_Mc2AxisGear` |
| `I_AxisRef` | `I_Mc2AxisRef` |
| `ST_MotionDefaults` | `ST_Mc2MotionDefaults` |
| `MotionTask`, `MotionTaskCollection` | `Mc2MotionTask`, `Mc2MotionTaskCollection` |
| `I_MotionTask`, `I_MotionTaskCollection`, `I_MotionJogTask`, `I_MotionPowerTask` | `I_Mc2MotionTask`, `I_Mc2MotionTaskCollection`, `I_Mc2MotionJogTask`, `I_Mc2MotionPowerTask` |
| `MoveAbsoluteTask`, `MoveRelativeTask`, `MoveVelocityTask`, `MoveModuloTask`, `MoveJogTask` | `Mc2MoveAbsoluteTask`, `Mc2MoveRelativeTask`, `Mc2MoveVelocityTask`, `Mc2MoveModuloTask`, `Mc2MoveJogTask` |
| `PowerTask`, `StopTask` | `Mc2PowerTask`, `Mc2StopTask` |
| `CoEResetTask`, `CoEExtendedResetTask`, `SoEResetTask` | `Mc2CoEResetTask`, `Mc2CoEExtendedResetTask`, `Mc2SoEResetTask` |
| `MotionHomeTask_SetZeroHere`, `MotionHomeTask_SetPosition`, `MotionHomeTask_SetPositionOffset`, `MotionHomeTask_CoESetEncoderOffset` | `Mc2MotionHomeTask_…` (same suffixes) |
| `MotionCoupleTask_Gearing` | `Mc2MotionCoupleTask_Gearing` |
| `MotionTask_AX5000AmpToNm` | `Mc2MotionTask_AX5000AmpToNm` |
| `ApplicationBaseMotionParameterNc2.TcGVL` | `ApplicationBaseMotionParameter.TcGVL` |
| `Events_Nc2.tmc` | `Events_MC.tmc` |

The technology-neutral interfaces (`I_AxisStatus`, `I_AxisDynamics`, `I_AxisEnable`, `I_AxisHome`, `I_AxisJog`, `I_AxisMove*`) and `ST_AxisPTP_HMI` kept their names — application code typed against those needs no change.

`Mc2Axis` also gained the `Direction` and `JogMode` settings properties and a touch-probe homing task in V2.0.0.

> **Careful:** `AxisPTP_HMI`, `Axis_TcEvents`, `I_Axis`, `I_Axis_PTP` and `I_MotionTaskCollection` were reintroduced
> in V2.1.0 as the *shared, generation-agnostic* types. The V1.x names are back, but `I_Axis` no longer means
> "the MC2 axis" — it means "any axis". V1.x code that used `AxisPTP_HMI` or `Axis_TcEvents` compiles again
> unchanged; V1.x code that used `I_Axis` to reach `AXIS_REF` or the MC2 settings must use `I_Mc2Axis`.

## Migrating from V2.0.x

V2.1.0 removed every motion-library type from the shared abstraction, so one HMI and one event-reaction
function block now serve both stacks. This is a **breaking change** for code that reached through the axis
interface to the NC.

| V2.0.x | V2.1.0 |
|--------|--------|
| `Mc2AxisPTP_HMI`, `Mc3AxisPTP_HMI` | `AxisPTP_HMI` (one shared FB, takes `I_Axis_PTP`) |
| `Mc2Axis_TcEvents`, `Mc3Axis_TcEvents` | `Axis_TcEvents` (one shared FB, takes `I_Axis`) |
| `myAxis.Axis.NcToPlc.Status.Coupled` | `myAxis.Coupled` |
| `myAxis.Axis.McToPlc.Std.AxisState = EAxisState.SynchronizedMotion` | `myAxis.SynchronizedMotion` |
| `myAxis.JogMode := E_JogMode.MC_JOGMODE_STANDARD_FAST` | `myAxis.JogMode := E_AxisJogMode.Fast` |
| `myAxis.JogMode := EJogMode.StandardFast` | `myAxis.JogMode := E_AxisJogMode.Fast` |
| `MotionTasks : I_Mc2MotionTaskCollection` / `I_Mc3MotionTaskCollection` | `MotionTasks : I_MotionTaskCollection` (read-only) |

What changed and why:

- **The raw `AXIS_REF` moved off the *shared* abstraction, not off `I_Mc2Axis` / `I_Mc3Axis`.** Those still
  extend `I_Mc2AxisRef` / `I_Mc3AxisRef`, so `myAxis.Axis` keeps working and raw NC access is unchanged. What
  changed is that the new `I_Axis` / `I_Axis_PTP` do not carry it — which is what let the HMI and event
  function blocks collapse from four to two.
- **`JogMode` moved from `I_Mc2Settings` / `I_Mc3Settings` to the shared `I_AxisSettings`** and is now a
  generation-neutral `E_AxisJogMode` (`Slow` / `Fast`) passed by value, with Get *and* Set. Previously it was a
  `REFERENCE TO` a library enum, written through the getter. This also fixes `I_Mc2Settings.JogMode` having been
  declared `REFERENCE TO Tc2_MC2.MC_Direction` instead of an `E_JogMode`.
- **`I_AxisStatus` gained `Coupled`, `SynchronizedMotion` and `Disabled`**, each mapped by the concrete axis from
  its own generation's status model, so consumers no longer decode `AxisState` themselves.
- **`MotionTasks` returns the read-only `I_MotionTaskCollection`.** `AddTask()` / `RemoveTaskByInstance()` stay on
  the generation-specific collection interfaces, which the axis uses internally.
- The motion-task layer is unchanged: tasks still reach the NC through `_Axis.Axis`.
