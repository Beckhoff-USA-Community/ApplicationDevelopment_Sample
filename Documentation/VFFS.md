# VFFS Demo — Components and Hierarchy

The VFFS (Vertical Form Fill Seal) demo is a sample application that models a packaging machine. It demonstrates how the `ApplicationBase` framework composes **one Machine Module**, **three Equipment Modules**, and several **Components** into a complete, PackML-compliant application.

## Module Hierarchy

```
VFFS  (MachineModule / PackMLModule)
├── Unwind  (EquipmentModule / PackMLModule)
│   └── UnwindAxis  (AxisPTP)
├── Sealer  (EquipmentModule / PackMLModule)
│   ├── SealerAxis  (AxisPTP)
│   └── SealBar  (CyclicComponent — custom)
└── PullWheels  (EquipmentModule / PackMLModule)
    ├── PullWheelLeftAxis   (AxisPTP — master)
    ├── PullWheelRightAxis  (SlaveAxisPTP — geared to left)
    └── PullWheelCylinder   (Cylinder)
```

## Machine Module — `VFFS`

`VFFS` is the top-level `PackMLModule`. It owns the three equipment modules, runs the production sequence state machine, and exposes the machine-level interface `I_VFFS`.

**Key responsibilities:**
- Orchestrates the production cycle (Unwind → Sealer → PullWheels → repeat)
- Tracks `PartsProduced`, `PartTime`, `SealingTime`, and `PullingTime`
- Manages a `RecipeManagement` instance holding the top-level `ST_Recipe`
- Suspends automatically when the sealer temperature leaves the valid range

**Production sequence (Execute state):**

| Step | Action |
|------|--------|
| 0 | Start Unwind axis at recipe velocity |
| 10 | Start Sealer (heat, close jaws, dwell) |
| 11 | Wait for Sealer completed; record seal time; reset Sealer |
| 20 | Start PullWheels (extend cylinder, pull, retract) |
| 21 | Wait for PullWheels completed; increment `PartsProduced`; go back to step 10 |

## Equipment Module — `Unwind`

Controls the film unwind axis. Implements `I_Unwind`.

| Component | Type | Purpose |
|-----------|------|---------|
| `UnwindAxis` | `AxisPTP` | Moves film at constant recipe velocity |

**Recipe — `ST_UnwindRecipe`:**

| Field | Type | Default |
|-------|------|---------|
| `Axis` | `ST_AxisRecipe` | Velocity 100, Accel/Decel/Jerk 1000 |

## Equipment Module — `Sealer`

Controls the jaw sealing station. Implements `I_Sealer`.

| Component | Type | Purpose |
|-----------|------|---------|
| `SealerAxis` | `AxisPTP` | Opens and closes the sealing jaws |
| `SealBar` | `SealBar` (custom `CyclicComponent`) | Simulates heater; exposes `Heat`, `SetTemperature`, `ActualTemperature`, `InTempRange` |

**Recipe — `ST_SealerRecipe`:**

| Field | Type | Default |
|-------|------|---------|
| `Axis` | `ST_AxisRecipe` | — |
| `Temp` | `LREAL` | 100 °C |
| `SealTime` | `LREAL` | 250 ms |
| `OpenPosition` | `LREAL` | 105 |
| `ClosedPosition` | `LREAL` | 200 |

**Suspension:** The `VFFS` machine transitions to *Suspended* whenever `SealBar.InTempRange = FALSE` and resumes once temperature returns to range.

## Equipment Module — `PullWheels`

Pulls the film by a fixed length per cycle using two geared axes and a pneumatic cylinder. Implements `I_PullWheels`.

| Component | Type | Purpose |
|-----------|------|---------|
| `PullWheelLeftAxis` | `AxisPTP` | Master drive axis |
| `PullWheelRightAxis` | `SlaveAxisPTP` | Slave axis, electronically geared to left |
| `PullWheelCylinder` | `Cylinder` | Engages / disengages the pull wheels |

**Starting sequence:** Extend cylinder → move axes by recipe `Length` → retract cylinder.

**Recipe — `ST_PullWheelsRecipe`:**

| Field | Type | Default |
|-------|------|---------|
| `Length` | `LREAL` | 100.0 mm |
| `LeftAxis` | `ST_AxisRecipe` | — |
| `RightAxis` | `ST_AxisRecipe` | — |

## Custom Component — `SealBar`

`SealBar EXTENDS CyclicComponent IMPLEMENTS I_SealBar`

A simulated thermal heater that demonstrates how to author a custom `CyclicComponent`.

- Increments `ActualTemperature` by 0.3 per 100 ms scan when `Heat = TRUE`
- Decrements by 0.1 per scan when `Heat = FALSE` (floor 20 °C)
- Sets `InTempRange = TRUE` when within ±1 °C of `SetTemperature`

## PackML State Machine

Every module in the VFFS demo extends `PackMLModule`, which wraps the framework `Statemachine` into the PackML v2022 state model.

**States used:** Idle → Starting → Execute → Completing → Completed → Stopping → Stopped → Aborting → Aborted → Clearing → (back to Stopped)

**Unit Modes:** Production, Maintenance, Manual. In Maintenance and Manual modes, sub-modules are exposed for independent HMI control.

## Recipe Structure

```
ST_Recipe
├── Sealer:    ST_SealerRecipe
│   └── Axis:  ST_AxisRecipe
├── Unwind:    ST_UnwindRecipe
│   └── Axis:  ST_AxisRecipe
└── PullWheels: ST_PullWheelsRecipe
    ├── LeftAxis:  ST_AxisRecipe
    └── RightAxis: ST_AxisRecipe
```

`ST_AxisRecipe` is a shared struct reused by every axis: `Velocity`, `Acceleration`, `Deceleration`, `Jerk`.

## Interfaces

| Interface | Implemented by | Key additions over base |
|-----------|---------------|------------------------|
| `I_VFFS` | `VFFS` | `PartsProduced`, `PartTime`, `SealingTime`, `PullingTime`, `Recipe`, `PackTags`, sub-module accessors |
| `I_Unwind` | `Unwind` | `Recipe`, `UnwindAxis` |
| `I_Sealer` | `Sealer` | `Recipe`, `SealerAxis`, `SealBar` |
| `I_PullWheels` | `PullWheels` | `Recipe`, `PullWheelLeftAxis`, `PullWheelRightAxis`, `PullWheelCylinder` |
| `I_SealBar` | `SealBar` | `Heat`, `SetTemperature`, `ActualTemperature`, `InTempRange` |

## Entry Point

`MAIN.TcPOU` instantiates `VFFS`, calls `VFFS.Initialize()` on the first scan, then calls `VFFS.CyclicLogic()` every 100 ms. A `GetSystemTreeJsonHmiVisitor` traverses the module tree to build the HMI symbol map.
