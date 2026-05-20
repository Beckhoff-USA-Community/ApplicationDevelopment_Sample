# ApplicationDevelopment_Sample

## Table of Contents

- [Intent](#intent)
  - [Why OOP and SOLID for Modern Machine Software](#why-oop-and-solid-for-modern-machine-software)
- [How to Use This Repository](#how-to-use-this-repository)
- [Repository Structure](#repository-structure)
- [Build & Run](#build--run)
- [Component-Module Hierarchy for Modern Machine Design](#component-module-hierarchy-for-modern-machine-design)
  - [Documentation](#documentation)
- [Disclaimer](#disclaimer)

## Intent

This repository demonstrates how a PLC application can be structured using **OOP and SOLID principles** in TwinCAT 3. It serves two purposes:

- Show how Beckhoff products can be used interconnectedly and how existing TwinCAT libraries can be functionally applied.
- Provide a structured foundation that enables **automatic HMI generation**, easy information aggregation once a project is organized and more.

This is an **open and collaborative repository**. Contributions, improvements, and adaptations are welcome. Before contributing, please familiarise yourself with **SOLID principles** and common **software design patterns** — the codebase is built on these foundations and all contributions are expected to follow them. Helpful resources include the [Design Patterns Catalog](https://refactoring.guru/design-patterns/catalog) on Refactoring.Guru and the [Design Patterns video series](https://www.youtube.com/@ChristopherOkhravi) by Christopher Okhravi on YouTube.

### Why OOP and SOLID for Modern Machine Software

1. **Maintainability** — Code organised into focused, single-responsibility classes is easier to read, debug, and extend. Changes to one part of the system do not ripple unpredictably through the rest.

2. **Reusability** — Well-defined components and interfaces can be lifted out of one project and dropped into another without modification, reducing duplication and the cost of starting new projects.

3. **Testability** — Isolated components with clear interfaces can be unit-tested independently, catching bugs early and giving confidence that individual building blocks behave correctly before they are integrated.

4. **Scalability** — A hierarchical module structure grows naturally with machine complexity. New equipment or functionality is added by composing existing building blocks rather than rewriting existing logic.

5. **Team collaboration** — Interfaces and abstractions create stable contracts between components, allowing multiple developers to work in parallel on different parts of a machine without stepping on each other's code.

## How to Use This Repository

**1. As a source of ideas** — Browse the code to see how OOP and SOLID principles apply to PLC development. Use it as a reference when designing your own architecture without taking any code directly.

**2. As a copy-template** — Take only what you need. Copy individual function blocks, components, or patterns into your own project and leave the rest behind. The framework is designed so that pieces work independently.

**3. As a complete framework** — Reference the entire `ApplicationBase` library in your TwinCAT project and build your application on top of it, using the module-component hierarchy as your foundation from day one. The code is fully open and can be freely changed or adapted to fit a wide variety of needs and project requirements. > **Warning:** See the [Disclaimer](#disclaimer) below — this code is provided as-is and must be validated for your specific application before use in production.

## Repository Structure

```
ApplicationDevelopment_Sample/
├── ApplicationBase/          # Reusable framework library (tspproj)
├── ApplicationDevelopment/   # Solution with applications and tests (sln)
│   ├── Template/             # Minimal pre-wired starting point
│   ├── VFFS/                 # Full demo — Vertical Form Fill Seal machine
│   └── UnitTests/            # Tests + mockups for ApplicationBase
└── Documentation/            # Topic-by-topic reference docs
```

**1. ApplicationBase**  — The core framework library. Referenced by other projects in this repository and by external TwinCAT projects that want to build on the component-module hierarchy. Contains all reusable function blocks, interfaces, and utilities.

**2. Unit Tests**  — A dedicated TwinCAT PLC project that tests the function blocks in `ApplicationBase`. Covers all major building blocks and serves as executable documentation of how each function block is intended to be used.

**3. VFFS Demo**  — A sample application modelling a Vertical Form Fill Seal (VFFS) packaging machine. Demonstrates how to apply the `ApplicationBase` framework to a realistic machine design, showing how modules, components, and state machines compose into a complete application.

**4. Template Project** — A minimal, pre-wired TwinCAT project to use as a starting point for new applications. Provides the scaffolding and references needed to build on `ApplicationBase` without having to set up the structure from scratch.

## Build & Run

This project requires **TwinCAT 3 XAE IDE** (Visual Studio extension, Windows-only).

- **Open the solution** — `ApplicationDevelopment/ApplicationDevelopment.sln`
- **Build** — right-click the solution in XAE and choose *Build*. Targets: TwinCAT RT or TwinCAT OS, Debug/Release, x86 / x64 / ARMV7-A / ARMV7-M / ARMV8-A.
- **Run** — activate the desired PLC task (`Template`, `VFFS`, or `UnitTests`), log in, and start the runtime on a controller or simulator.
- **Run the unit tests** — start the `UnitTests` task; `UnitTests/MAIN.TcPOU` orchestrates every test. Inspect results in the global `GlobalTestSuite.TestSuite` via the TwinCAT online view. To run a single test, comment out the other test function calls in `MAIN.TcPOU`.

## Component-Module Hierarchy for Modern Machine Design

The application sample code models a machine as a tree of **Modules** and **Components** — from a top-level `MachineModule` down to individual `Component` leaves. Modules coordinate initialization and cyclic execution of everything they contain. Tree-wide operations (reset, mode change, HMI generation) are applied via the **Visitor pattern**, keeping operations decoupled from the objects they act on.

### Documentation

Full reference docs live in [`Documentation/`](Documentation/README.md). Highlights below, grouped by topic.

**Application examples**

| Topic | Description |
|-------|-------------|
| [Template Project](Documentation/Template.md) | Minimal pre-wired starting point — `Machine` FB extending `PackMLModule`, HMI, recipe loading, and events, all wired and ready to extend |
| [VFFS — Demo Application](Documentation/VFFS.md) | Vertical Form Fill Seal packaging machine — the demo project where the full framework is applied end-to-end: module/component hierarchy, PackML state machine, events, visitors, and HMI |

**Framework core**

| Topic | Description |
|-------|-------------|
| [Component & CyclicComponent](Documentation/Component.md) | Base classes for all building blocks |
| [Module](Documentation/Module.md) | Container hierarchy — `EquipmentModule` and `MachineModule` |
| [Statemachine](Documentation/Statemachine.md) | Generic indexed state machine and `State` base class |
| [Visitors](Documentation/Visitors.md) | Tree-traversal operations (reset, enable, mode change, HMI, …) |
| [Interfaces](Documentation/Interfaces.md) | Standalone interface reference (`I_Base`, `I_Enablable`, `I_TaskResult`, …) |
| [Collections & Buffers](Documentation/Collections.md) | `AnyBuffer`, `ComponentCollection`, `Collection` |
| [Utilities](Documentation/Utilities.md) | `ForcibleBool/Int`, `AnalogScale`, `Stopwatch`, `RecipeManagement` |

**I/O components**

| Topic | Description |
|-------|-------------|
| [Digital I/O](Documentation/Digital.md) | `DigitalInput_NO/NC`, `DigitalOutput`, Combiner, Debounce |
| [Analog I/O](Documentation/Analog.md) | `AnalogInput`, `AnalogOutput`, `AnalogScale` |
| [Cylinder](Documentation/Cylinder.md) | Double-acting cylinder controller with state machine and fault detection |
| [Safety](Documentation/Safety.md) | `SafetyBase`, `SafetyAndOrFB`, `SafetyModule`, `SafetyResetPulse` |

**Communication**

| Topic | Description |
|-------|-------------|
| [EtherCAT](Documentation/EtherCAT.md) | `EtherCatMaster`, `EtherCatIoDevice` |
| [CoE](Documentation/CoE.md) | `CoeDevice`, `NullCoeDevice` — SDO read/write for EtherCAT slaves |
| [ADS](Documentation/ADS.md) | `AdsReadWrite` — ADS read/write by index or symbol name |
| [Serial](Documentation/Serial.md) | `SerialByteConnection`, `SerialStringConnection`, line control variants |
| [TCP/IP](Documentation/TcpIp.md) | `TcpIpConnection`, `TcpIpCommandResultFilter` |

**Operator interaction**

| Topic | Description |
|-------|-------------|
| [Events](Documentation/Events.md) | `TcEventClass`, `I_EventClass`, `I_EventReaction`, `Trace` logging utility |
| [HMI](Documentation/HMI.md) | `HmiFunction`, `Button`, `PermissiveInterlock` |

## Disclaimer

All sample code provided by Beckhoff Automation LLC are for illustrative purposes only and are provided "as is" and without any warranties, express or implied. Actual implementations in applications will vary significantly. Beckhoff Automation LLC shall have no liability for, and does not waive any rights in relation to, any code samples that it provides or the use of such code samples for any purpose.
