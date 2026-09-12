# Changelog

Version history of the `ApplicationBase` library. The current version is published at runtime through
`Global_Version.stLibVersion_ApplicationBase` and `F_GetVersion()`.

## 2.2.0

**Requires TwinCAT 3.1.4026.27 and `Tc2_EtherCAT` 3.8.2.0 or newer.** `EtherCatMaster` now reads the slave states with
`FB_EcGetAllExtSlaveStates` into `ST_EcExtendedSlaveState`. Neither exists in older `Tc2_EtherCAT` releases, so the
library does not compile against them. The solution was upgraded to 4026.27 and every pinned library copy was
refreshed to the versions shipped with that build (`Tc2_EtherCAT` 3.8.2.0 among them).

**Breaking change — `I_EcIoDevice.State` changed type.** It is now `REFERENCE TO ST_EcExtendedSlaveState` instead of
`REFERENCE TO ST_EcSlaveState`. Application code that stored the reference or declared its own
`EtherCatIoDevice` with a `ST_EcSlaveState` has to be retyped; the `deviceState` / `linkState` words and every
`BOOL` property of `I_EcIoDevice` are unchanged.

**EtherCAT diagnostics scale.** The per-cycle work of `EtherCatMaster` is either constant or bounded by
`SLAVES_PER_CYCLE`. Verified target: 2500 slaves on a 10 ms task. See
[Documentation/EtherCAT.md](Documentation/EtherCAT.md#changes-in-220).

### Added

- `I_CoeTransfer` and `I_CoeTransferScheduler` — scheduling contract for CoE transfers. `CoeDevice` implements
  `I_CoeTransfer` and gained a `Scheduler` property; `EtherCatMaster` implements `I_CoeTransferScheduler` and
  passes itself to every `EtherCatIoDevice` through the new `CoeScheduler` setter. A device with a scheduler
  registers itself in `Read` / `Write` and is dropped from the serviced set when `Busy` falls
- `E_EcErrorSource` (`None`, `Master`, `Slave`, `Ads`, `Sequence`) and `I_EtherCatMasterDiagnostic.ErrorSource` —
  says what `ErrorId` means: `DevState` word, configured slave index, function block error id, or the step
  number / offending count of the sequence itself
- `I_EtherCatMasterDiagnostic.GetSlaveIndexByAddr(Addr)` — configured slave index for an EtherCAT address,
  -1 if unknown. O(1) for addresses 1001 .. 1001 + `MAX_EC_SLAVES`, linear for user-assigned addresses
- `EtherCatParameter.SLAVES_PER_CYCLE` (64) — slaves handled per cycle by the multi-cycle passes
- `EtherCatParameter.ADS_WAIT_CYCLES` (1000) and `INIT_WAIT_CYCLES` (60000) — cycle watchdogs on every wait
  step of the diagnostic and of `Initialize()`; a timeout aborts with `ErrorSource = Ads`, `ErrorId = 16#745`
- `EtherCatMaster_TEST` with `EtherCatMaster_Mockup` and `EtherCatEventProvider_Mockup`; the `UnitTests`
  project now references `Tc2_EtherCAT`

### Changed

- `EtherCatMaster` reads all slave states with `FB_EcGetAllExtSlaveStates`; `SlavesState`, the `State`
  property of `EtherCatIoDevice` and `I_EcIoDevice.State` are typed `ST_EcExtendedSlaveState`. The master and
  the io device still decode only `deviceState` and `linkState`; the extended fields are exposed to the
  application through the `State` reference
- The per-slave `CyclicLogic()` loop is gone. `EtherCatMaster` services only the CoE transfers in flight;
  `EtherCatIoDevice.CyclicLogic()` services its own `CoeDevice` only when no `CoeScheduler` is attached
- Address-to-index map replaces the linear scans of the sync unit assignment and of `GetIoDeviceByAddr`;
  sync unit name resolution is O(n) instead of O(n²)
- Every O(n) pass — classification, evaluation, sync unit resolution, io device wiring — runs at
  `SLAVES_PER_CYCLE` slaves per cycle. `Busy` therefore stays TRUE for up to n / `SLAVES_PER_CYCLE` cycles and
  `Error` of a change lands that many cycles after the trigger
- ADS reads are sized to the configured slave count. The topology is read once in `Initialize()` and again
  only when a slave needs a diagnostic; a clean network ends the diagnostic after the classification pass
- The diagnostic is triggered by a change of `ChangeCount`, of `SlaveCount` while it differs from
  `CfgSlaveCount`, or of the frame working counter state
- `ErrorId` keeps the raw code of its `ErrorSource`. The placeholder value 999 is gone and ADS errors keep the
  function block error id
- `Initialize()` fails with `ErrorSource = Sequence` when more slaves are configured than `MAX_EC_SLAVES + 1`
- `GetIoDeviceByAddr` and `GetIoDeviceByName` log a message when they fall back to the null device
- `CoeDevice.Error` / `ErrorId` describe the last started transfer only; `Read` / `Write` without an `AdsAddr`
  set `Error` with `ErrorId = 16#70B` instead of returning silently
- Solution upgraded to TwinCAT 3.1.4026.27; pinned copies of the Beckhoff libraries refreshed accordingly

### Fixed

- The master TcEvent was raised every cycle instead of once per `DevState` change
- `DeviceError`, `Disabled`, `InvalidVPRS` and `InitCmdError` of `EtherCatIoDevice` were always FALSE
- `CoeDevice.Write` used the SDO read function block
- The sync unit re-read and re-matched its slaves on every working counter recovery and logged an error each
  pass for an empty sync unit
- `Reset()` did not clear the frame state
- Init trace texts said "mapped" for "not mapped"

## 2.1.0

**Breaking change — the shared axis abstraction is now free of motion-library types.** See
[Documentation/Motion.md](Documentation/Motion.md#migrating-from-v20x) for the migration table.

`I_Mc2Axis` and `I_Mc3Axis` no longer expose `AXIS_REF` or any `Tc2_MC2` / `Tc3_Mc3Ptp` enum. That leak was the
only reason the HMI and event-reaction function blocks existed once per motion generation, so they have been
merged into a single shared pair.

### Added

- `Mc2AxisParameterLoader` and `Mc3AxisParameterLoader` — the NC parameter read is now its own
  `I_Initializable` component instead of a state machine inside the axis. The MC3 loader is table-driven:
  `AddParameter(ParameterId, ADR(Target))` per value, so reading one more parameter is one more line rather
  than an edit to a `CASE` ladder. The MC2 loader has no table because `MC_ReadParameterSet` reads the whole
  `ST_AxisParameterSet` in one call
- `ST_Mc3ParameterBinding` — one entry in the MC3 loader's table
- `I_AxisGear` — generation-neutral gearing role (`GearIn()`, `GearOut()`). `I_Mc2AxisGear` and
  `I_Mc3AxisGear` extend it, so couple/decouple is discoverable from any axis with
  `__QUERYINTERFACE(axis, gearing)` rather than only from the `I_Mc?SlaveAxis` subtype
- `I_Mc3AxisGear.GearOut()` and `Mc3SlaveAxisPTP.GearOut()` — MC3 could couple but never decouple.
  `Tc3_Mc3Ptp` has no `MC_GearOut`, so `Mc3MotionCoupleTask_Gearing` ends the synchronized motion with
  `MC_Halt`
- `ApplicationBaseMotionParameter.MaxAxisParameters` (10) — generic bound on `Mc3AxisParameterLoader`
- `I_Axis` and `I_Axis_PTP` — generation-agnostic axis abstractions in `Motion/Interface/`. `I_Mc2Axis` and
  `I_Mc3Axis` now extend `I_Axis`; `I_Mc2Axis_PTP` and `I_Mc3Axis_PTP` extend `I_Axis_PTP`
- `I_AxisSettings` — holds the generation-neutral `JogMode`
- `I_MotionTaskCollection` — read-only view (`Busy`, `Error`, `ErrorId`, `ErrorTask`) on a task collection;
  `I_Mc2MotionTaskCollection` and `I_Mc3MotionTaskCollection` now extend it
- `E_AxisJogMode` (`Slow`, `Fast`) — generation-neutral jog mode in `Motion/DUT/`
- `AxisPTP_HMI` and `Axis_TcEvents` in `Motion/` — one operator faceplate and one event reaction serving both
  stacks, typed on `I_Axis_PTP` / `I_Axis`
- `I_AxisStatus` gained `Coupled`, `SynchronizedMotion` and `Disabled`

### Changed

- `I_Axis` and `I_Axis_PTP` do not expose `AXIS_REF`. `I_Mc2Axis` / `I_Mc3Axis` still extend
  `I_Mc2AxisRef` / `I_Mc3AxisRef`, so raw NC access through `myAxis.Axis` is unchanged — it is simply no
  longer reachable from the generation-agnostic abstraction
- `JogMode` moved from `I_Mc2Settings` / `I_Mc3Settings` to `I_AxisSettings`, is now an `E_AxisJogMode` passed by
  value with Get and Set, and each axis maps it onto its own library enum
- `Mc2Axis.MotionTasks` / `Mc3Axis.MotionTasks` return `I_MotionTaskCollection`. `AddTask()` and
  `RemoveTaskByInstance()` remain on the generation-specific interfaces for internal use
- VFFS was updated to the shared `AxisPTP_HMI` / `Axis_TcEvents`
- `Mc2Axis.Initialize()` / `Mc3Axis.Initialize()` no longer contain the read sequence; they delegate to the
  parameter loader and copy the result into the dynamics properties
- `Initialize()` now has exactly one driver. A registered axis is driven by its parent module's
  `Initializer`; an axis with no parent self-initializes from `CyclicLogic()`. Previously both paths were
  live, and only the `RETURN` in `MAIN` before `CyclicLogic()` kept them from overlapping
- **Every PTP command now refuses while the axis is coupled.** `MoveAbsolute`, `MoveRelative`,
  `MoveVelocity`, `MoveModulo`, `Jog` and `Home` check `I_AxisStatus.Coupled` and return without
  dispatching. `Stop` is deliberately still allowed, since halting the slave is how MC3 decouples it.
  The guard sits on `Mc2AxisPTP` / `Mc3AxisPTP` rather than on the slave subclass, so the base contract
  holds for every subtype and `Mc?SlaveAxisPTP` no longer narrows it

### Removed

- `Mc2AxisPTP_HMI`, `Mc3AxisPTP_HMI`, `Mc2Axis_TcEvents`, `Mc3Axis_TcEvents` — replaced by the shared
  `AxisPTP_HMI` and `Axis_TcEvents`

### Fixed

- The four `Move*` methods promised a result and always delivered `FALSE`: `MoveRelative`, `MoveVelocity`
  and `MoveModulo` were declared `: BOOL` but never assigned a return value, and `MoveAbsolute` had no
  return at all. All four now return `TRUE` when the command is dispatched and `FALSE` when refused.
  `I_AxisMoveAbsolute.MoveAbsolute` gained the `: BOOL` return to match its three siblings

- `I_Mc2Settings.JogMode` was declared `REFERENCE TO Tc2_MC2.MC_Direction` while `Mc2Axis` backed it with an
  `E_JogMode` and the HMI assigned `E_JogMode` values to it. The property was replaced by the typed
  `E_AxisJogMode` on `I_AxisSettings`

## 2.0.0

**Breaking change — the motion stack was restructured.** See [Documentation/Motion.md](Documentation/Motion.md)
for the full reference and the [rename table](Documentation/Motion.md#migrating-from-v1x).

### Added — MC3 axis stack

A second, parallel axis stack built on the current Beckhoff motion API (`Tc3_Mc3Base` / `Tc3_Mc3Ptp`),
sitting behind the same technology-neutral interfaces as the MC2 stack:

- `Mc3Axis`, `Mc3AxisPTP`, `Mc3SlaveAxisPTP` — component, point-to-point and geared-slave axes
- `Mc3AxisPTP_HMI` — operator faceplate with a permissive per command
- `Mc3Axis_TcEvents` — raises `E_McAxis.AxisError` with the failing task name and error id
- `I_Mc3Axis`, `I_Mc3Axis_PTP`, `I_Mc3SlaveAxis`, `I_Mc3AxisGear`, `I_Mc3Settings`, `I_Mc3AxisRef`
- MC3 motion tasks under `_Internal/MotionTasks/MC3/` — power, stop, jog, move absolute/relative/velocity/modulo,
  gearing, set-position and set-zero-here homing, CoE reset — plus `Mc3MotionTask`, `Mc3MotionTaskCollection`
  and `ST_Mc3MotionDefaults`
- Library references `Tc3_Mc3Base` and `Tc3_Mc3Ptp`

MC3 gearing uses a single `Master` with `RatioNumerator` / `RatioDenominator` and exposes `InGear`,
`MasterHasError`, `ReactionToMasterError` and `ReactionToSlaveError`.

### Changed — MC2 axis

- Every pre-existing motion type was renamed with an `Mc2` prefix and moved under `Motion/MC2/` and
  `_Internal/MotionTasks/MC2/` — `Axis` → `Mc2Axis`, `AxisPTP` → `Mc2AxisPTP`, `SlaveAxisPTP` → `Mc2SlaveAxisPTP`,
  `I_Axis` → `I_Mc2Axis`, `MotionTask` → `Mc2MotionTask`, and so on
- `Mc2Axis` gained the `Direction` and `JogMode` settings properties, collected together with `BufferMode`,
  `MoveOptions` and `AxisParameterSet` under the new `I_Mc2Settings` interface
- New `Mc2MotionHomeTask_TouchProbe` homing task
- `ApplicationBaseMotionParameterNc2.TcGVL` renamed to `ApplicationBaseMotionParameter.TcGVL`
- `Events_Nc2.tmc` renamed to `Events_MC.tmc`
- Library upgrades: `Tc2_MC2_Camming` and `Tc3_MC2_AdvancedHoming` added, `Tc2_NC` updated

The technology-neutral interfaces — `I_AxisStatus`, `I_AxisDynamics`, `I_AxisEnable`, `I_AxisHome`,
`I_AxisJog`, `I_AxisMoveAbsolute/Relative/Velocity/Modulo` — and `ST_AxisPTP_HMI` kept their names, so
application code typed against those is unaffected.

### Changed — VFFS demo

The VFFS demo application was migrated to MC3: `Unwind`, `Sealer` and `PullWheels` now use `Mc3AxisPTP`,
`Mc3SlaveAxisPTP`, `Mc3AxisPTP_HMI` and `Mc3Axis_TcEvents`. The unit tests continue to cover the MC2 axis.

## 1.1.0

- Added motion maths helpers under `Utilities/Motion/` — `ATAN2`, `CalculateHypotenuseAngle`,
  `CalculateMotionDynamics`, `CamSmoothDerivatives`, `XPlanar6DMoverToWorldTransform`
- Library version upgrades

## 1.0.8

- `I_Base` added to the component and module collections

## 1.0.7

- Sample disclaimer added

## 1.0.5

- Removed `FB_DynMem_Buffer` usage

## 1.0.4

- TwinCAT 4026.24; motion tasks are now called cyclically

## 1.0.3

- Added `LocalLogger`; `Trace` now also implements `I_TraceLogger`

## 1.0.2

- `Initializer` exposes `NotInitialized` as a debugging aid

## 1.0.1

- Upgrade to TwinCAT 4026.24; removed a duplicate initialization registration

## 1.0.0

- Initial reference library
