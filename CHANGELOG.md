# Changelog

Version history of the `ApplicationBase` library. The current version is published at runtime through
`Global_Version.stLibVersion_ApplicationBase` and `F_GetVersion()`.

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
