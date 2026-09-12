# Documentation

Reference docs for the `ApplicationBase` framework and the sample applications built on top of it. Start at the [root README](../README.md) for an overview of the repository, or the [changelog](../CHANGELOG.md) for what changed between library versions.

## Application Examples

| Topic | Description |
|-------|-------------|
| [Template Project](Template.md) | Minimal pre-wired starting point — `Machine` FB extending `PackMLModule`, HMI, recipe loading, and events, all wired and ready to extend |
| [VFFS — Demo Application](VFFS.md) | Vertical Form Fill Seal packaging machine — the demo project where the full framework is applied end-to-end |

## Framework Core

| Topic | Description |
|-------|-------------|
| [Component & CyclicComponent](Component.md) | Base classes for all building blocks |
| [Module](Module.md) | Container hierarchy — `EquipmentModule` and `MachineModule` |
| [Statemachine](Statemachine.md) | Generic indexed state machine and `State` base class |
| [Visitors](Visitors.md) | Tree-traversal operations (reset, enable, mode change, HMI, …) |
| [Interfaces](Interfaces.md) | Standalone interface reference (`I_Base`, `I_Enablable`, `I_TaskResult`, …) |
| [Collections & Buffers](Collections.md) | `AnyBuffer`, `ComponentCollection`, `Collection` |
| [Utilities](Utilities.md) | `ForcibleBool/Int`, `AnalogScale`, `Stopwatch`, `RecipeManagement` |

## I/O Components

| Topic | Description |
|-------|-------------|
| [Digital I/O](Digital.md) | `DigitalInput_NO/NC`, `DigitalOutput`, Combiner, Debounce |
| [Analog I/O](Analog.md) | `AnalogInput`, `AnalogOutput`, `AnalogScale` |
| [Cylinder](Cylinder.md) | Double-acting cylinder controller with state machine and fault detection |
| [Safety](Safety.md) | `SafetyBase`, `SafetyAndOrFB`, `SafetyModule`, `SafetyResetPulse` |

## Motion

| Topic | Description |
|-------|-------------|
| [Motion](Motion.md) | MC2 and MC3 axis stacks — `Mc3AxisPTP`, `Mc2AxisPTP`, slave/geared axes, parameter loaders, motion tasks, the shared `AxisPTP_HMI` / `Axis_TcEvents`, the generation-neutral `I_Axis` / `I_Axis_PTP` / `I_AxisGear` interfaces, plus the V1.x and V2.0.x migration tables |

## Communication

| Topic | Description |
|-------|-------------|
| [EtherCAT](EtherCAT.md) | `EtherCatMaster`, `EtherCatIoDevice` — diagnostics that scale to 2500 slaves; requires TwinCAT 3.1.4026.27 / `Tc2_EtherCAT` 3.8.2.0+ |
| [CoE](CoE.md) | `CoeDevice`, `NullCoeDevice`, `I_CoeTransfer` / `I_CoeTransferScheduler` — SDO read/write for EtherCAT slaves |
| [ADS](ADS.md) | `AdsReadWrite` — ADS read/write by index or symbol name |
| [Serial](Serial.md) | `SerialByteConnection`, `SerialStringConnection`, line control variants |
| [TCP/IP](TcpIp.md) | `TcpIpConnection`, `TcpIpCommandResultFilter` |

## Operator Interaction

| Topic | Description |
|-------|-------------|
| [Events](Events.md) | `TcEventClass`, `I_EventClass`, `I_EventReaction`, `Trace` logging utility |
| [HMI](HMI.md) | `HmiFunction`, `Button`, `PermissiveInterlock` |
