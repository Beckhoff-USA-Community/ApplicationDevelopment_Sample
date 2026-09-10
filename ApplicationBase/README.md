# ApplicationBase

The reusable framework library that backs every sample project in this repository. It defines the **Component / Module hierarchy**, the **PackML-aligned state machine**, **visitors** for tree-wide operations, and a set of **I/O, communication, and utility components** ready to drop into a TwinCAT application.

This is a **TwinCAT library project** (`.tspproj`). It is not run directly — it is referenced by application projects (`Template`, `VFFS`, or your own) which compose its building blocks into a machine.

**Current version: 2.0.0** — published at runtime through `Global_Version.stLibVersion_ApplicationBase` and `F_GetVersion()`. V2.0.0 adds the MC3 axis stack and renames the existing motion types with an `Mc2` prefix; see the [changelog](../CHANGELOG.md) and the [migration notes](../Documentation/Motion.md#migrating-from-v1x).

## Usage

Open the project in TwinCAT XAE and build the library, or reference the released library from your own TwinCAT project. The sibling [`ApplicationDevelopment.sln`](../ApplicationDevelopment) already references this library and shows how to build on top of it.

## Where to Start

| If you want to … | Read |
|------------------|------|
| Understand the building blocks | [Component & CyclicComponent](../Documentation/Component.md) |
| Compose components into machines | [Module](../Documentation/Module.md) |
| Implement state-driven behaviour | [Statemachine](../Documentation/Statemachine.md) |
| Apply tree-wide operations | [Visitors](../Documentation/Visitors.md) |
| Control axes (MC2 or MC3) | [Motion](../Documentation/Motion.md) |
| See the full doc index | [Documentation/](../Documentation/README.md) |

## Folder Layout

| Folder | Purpose |
|--------|---------|
| `Component/` | Abstract `Component` and `CyclicComponent` base FBs |
| `Module/` | `Module`, `EquipmentModule`, `MachineModule`, `CyclicRunner`, `Initializer` |
| `Components/` | Concrete components — Digital, Analog, Cylinder, ADS, … |
| `Motion/` | `Mc2Axis` / `Mc3Axis` families — PTP, geared slave, HMI faceplate, events |
| `Kinematic Group/` | `KinematicGroup` and its events |
| `Statemachine/` | Generic state machine with mode control |
| `Modes/` | `ModeControl` operating mode management |
| `Tracing/` | Event logging and diagnostics |
| `Utilities/` | `AnyBuffer`, `Force`, `Stopwatch`, `RecipeManagement`, motion maths helpers, … |
| `Visitors/` | Tree-traversal implementations (visitor pattern) |
| `_Interface/` | All `I_*` interface definitions |
| `_Internal/` | Implementation details not meant for direct use — e.g. the MC2 / MC3 motion tasks |

## Global Configuration

Defaults live in `ApplicationBaseParameter.TcGVL`:

- `MaxCountInCollections`: 50 — upper bound for all collections, runners, and initializers
- `EnableAdsLogger`: TRUE
- `EnableTcEventLogging`: FALSE
- `TraceLevel`: Verbose
- `ClearingTimeout`: 2 s

Motion defaults live in `ApplicationBaseMotionParameter.TcGVL`:

- `MaxMotionTasks`: 10 — upper bound for the motion task collection owned by each axis
