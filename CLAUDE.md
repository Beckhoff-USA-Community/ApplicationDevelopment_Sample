# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **TwinCAT 3 PLC application** — a component-based automation framework for Beckhoff TwinCAT 3 real-time controllers, written in IEC 61131-3 Structured Text (ST). It serves as an educational sample demonstrating professional PLC development patterns.

## Build & Development

This project requires **TwinCAT 3 XAE IDE** (Visual Studio extension, Windows-only).

- **Open solution**: `ApplicationDevelopment/ApplicationDevelopment.sln`
- **Build**: Right-click solution → Build in TwinCAT XAE IDE
- **Target**: TwinCAT RT or TwinCAT OS in Debug/Release for x86, x64, ARMV7-A, ARMV7-M, or ARMV8-A

## Running Tests

Tests run inside the TwinCAT PLC runtime — there is no CLI test runner.

- **Run all tests**: Download and run the project on a TwinCAT controller or simulator; `UnitTests/MAIN.TcPOU` orchestrates all test calls
- **Run a single test**: Comment out all other test function calls in `MAIN.TcPOU`, leaving only the target test (e.g., `Component_TEST()`)
- **Test results**: Inspect the global `GlobalTestSuite.TestSuite` object at runtime via the TwinCAT online view

## Solution Structure

```
ApplicationDevelopment_Sample/
├── ApplicationBase/          # Reusable framework library (tspproj)
│   └── ApplicationBase/      # Framework source (components, modules, utilities)
└── ApplicationDevelopment/   # Applications and unit tests (sln + tsproj)
    ├── UnitTests/            # PLC project with all test POUs and mockups
    ├── Template/             # Minimal pre-wired starting point for new projects
    └── VFFS/                 # Full demo application (Vertical Form Fill Seal machine)
```

**Four projects:**
- `ApplicationBase` — the reusable framework (library project)
- `UnitTests` (inside `ApplicationDevelopment`) — exercises the framework with 26+ test modules
- `Template` (inside `ApplicationDevelopment`) — minimal pre-wired starting point for new applications
- `VFFS` (inside `ApplicationDevelopment`) — complete demo application modelling a packaging machine

## Architecture

### Core Concepts

The framework uses a **hierarchical, interface-driven component model**:

- **`Component`** — base unit; implements `I_Component`. All reusable building blocks extend this.
- **`CyclicComponent`** — extends `Component` with `I_Cyclic`; override `CyclicLogic()` for per-scan work.
- **`Module`** — aggregates components and sub-modules; implements `I_Module`, `I_Cyclic`, `I_Initializable`. Owns a `ComponentCollection` and `ModuleCollection`.
- **`CyclicRunner<MaxSize>`** — manages cyclic dispatch of registered `I_Cyclic` objects within a module.
- **`Initializer`** — manages ordered initialization sequences of `I_Initializable` objects.
- **`Statemachine<InitialState>`** — generic state machine FB extending `CyclicComponent`.

### Key Subsystems in `ApplicationBase`

| Folder | Purpose |
|--------|---------|
| `Component/` | Abstract `Component` and `CyclicComponent` base FBs |
| `Module/` | `Module`, `EquipmentModule`, `MachineModule`, `CyclicRunner`, `Initializer` |
| `Components/Digital/` | `DigitalInput_NC`, `DigitalInput_NO`, `DigitalOutput` |
| `Components/Analog/` | Analog I/O components |
| `Components/Cylinder/` | Pneumatic/hydraulic cylinder control |
| `Components/ADS/` | ADS (Automation Device Specification) communication |
| `Motion/` | Axis components — `Motion/MC2/` (`Mc2Axis`, `Mc2AxisPTP`, `Mc2SlaveAxisPTP`) and `Motion/MC3/` (`Mc3Axis`, `Mc3AxisPTP`, `Mc3SlaveAxisPTP`), behind shared `Motion/Interface/` axis interfaces (`I_Axis`, `I_Axis_PTP`, …). The generation-agnostic `AxisPTP_HMI` and `Axis_TcEvents` sit directly in `Motion/` |
| `_Internal/MotionTasks/` | Motion tasks driving the axes — `MC2/` and `MC3/` variants (move, power, home, reset, gearing) |
| `Kinematic Group/` | `KinematicGroup` and its events |
| `Statemachine/` | Generic state machine with mode control |
| `Modes/` | `ModeControl` operating mode management |
| `Tracing/` | Event logging and diagnostics |
| `Utilities/AnyBuffer/` | Generic queue/buffer (`Push`/`Pop`/`Peek`, left/right ends) |
| `Utilities/Force/` | Variable forcing utilities |
| `Visitors/` | Tree-traversal implementations (visitor pattern over component/module trees) |
| `_Interface/` | All `I_*` interface definitions |

### Visitor Pattern

Operations on the component/module hierarchy (reset, mode change, name enumeration) are implemented as visitors that implement `VisitComponent()` / `VisitModule()`. This avoids modifying the visited objects.

### Global Configuration

`ApplicationBase/ApplicationBaseParameter.TcGVL`:
- `MaxCountInCollections`: 50 — upper bound for all collections, runners, and initializers
- `EnableAdsLogger`: TRUE
- `EnableTcEventLogging`: FALSE
- `TraceLevel`: Verbose
- `ClearingTimeout`: 2 s

`ApplicationBase/ApplicationBaseMotionParameter.TcGVL`:
- `MaxMotionTasks`: 10 — upper bound for the motion task collection owned by each axis
- `MaxAxisParameters`: 10 — upper bound for the NC parameter table each `Mc3Axis` pre-loads

## Library Version

The `ApplicationBase` version lives in `Version/Global_Version.TcGVL` and `Project Information/F_GetVersion.TcPOU` — both are generated from the project properties in XAE, so bump the version there rather than editing the files by hand. Record notable changes in the root `CHANGELOG.md`.

Current version: **2.1.0**. V2.0.0 added the MC3 axis stack and renamed all pre-existing motion types with an `Mc2` prefix (breaking change). V2.1.0 removed `AXIS_REF` and the motion-library enums from the shared axis abstraction and merged the per-generation HMI/event function blocks into `AxisPTP_HMI` and `Axis_TcEvents` (breaking change) — see `Documentation/Motion.md`.

## Naming Conventions

| Convention | Pattern | Example |
|------------|---------|---------|
| Interfaces | `I_` prefix | `I_Component`, `I_Cyclic` |
| Structs | `ST_` prefix | `ST_Recipe` |
| Private variables | `_` prefix | `_Name`, `_IsInitialized` |
| Test functions | `_TEST` suffix | `Component_TEST`, `Module_TEST` |
| Global variable lists | `TcGVL` files | `ApplicationBaseParameter.TcGVL` |

## Test Authoring Pattern

```pascal
// Each test function guards execution:
IF NOT TestSuite.Test(__POUNAME()).ExecuteTest() THEN RETURN; END_IF

// Assertions:
TestSuite.AssertEqual(actual, expected, 'message');
```

Mockup objects for components and modules live in `UnitTests/_Mockups/`.

## Task Configuration (UnitTests)

- Task name: `UnitTests`, Priority: 20, Cycle time: 100 ms, AMS Port: 350
