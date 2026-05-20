# ApplicationBase

The reusable framework library that backs every sample project in this repository. It defines the **Component / Module hierarchy**, the **PackML-aligned state machine**, **visitors** for tree-wide operations, and a set of **I/O, communication, and utility components** ready to drop into a TwinCAT application.

This is a **TwinCAT library project** (`.tspproj`). It is not run directly — it is referenced by application projects (`Template`, `VFFS`, or your own) which compose its building blocks into a machine.

## Usage

Open the project in TwinCAT XAE and build the library, or reference the released library from your own TwinCAT project. The sibling [`ApplicationDevelopment.sln`](../ApplicationDevelopment) already references this library and shows how to build on top of it.

## Where to Start

| If you want to … | Read |
|------------------|------|
| Understand the building blocks | [Component & CyclicComponent](../Documentation/Component.md) |
| Compose components into machines | [Module](../Documentation/Module.md) |
| Implement state-driven behaviour | [Statemachine](../Documentation/Statemachine.md) |
| Apply tree-wide operations | [Visitors](../Documentation/Visitors.md) |
| See the full doc index | [Documentation/](../Documentation/README.md) |

## Folder Layout

| Folder | Purpose |
|--------|---------|
| `Component/` | Abstract `Component` and `CyclicComponent` base FBs |
| `Module/` | `Module`, `EquipmentModule`, `MachineModule`, `CyclicRunner`, `Initializer` |
| `Components/` | Concrete components — Digital, Analog, Cylinder, ADS, … |
| `Statemachine/` | Generic state machine with mode control |
| `Modes/` | `ModeControl` operating mode management |
| `Tracing/` | Event logging and diagnostics |
| `Utilities/` | `AnyBuffer`, `Force`, `Stopwatch`, `RecipeManagement`, … |
| `Visitors/` | Tree-traversal implementations (visitor pattern) |
| `_Interface/` | All `I_*` interface definitions |

## Global Configuration

Defaults live in `ApplicationBaseParameter.TcGVL`:

- `MaxCountInCollections`: 50 — upper bound for all collections, runners, and initializers
- `EnableAdsLogger`: TRUE
- `TraceLevel`: Verbose
- `ClearingTimeout`: 2 s
