# Template Project

The Template project is a minimal, pre-wired TwinCAT application that demonstrates how to connect the `ApplicationBase` framework into a working starting point. It provides the scaffold — `PackMLModule`, HMI, recipe loading, and events — all wired and ready to extend.

## Entry Point

`MAIN.TcPOU` instantiates `Machine('Machine')`, calls `Machine.Initialize()` on the first scan until initialization completes, then calls `Machine.CyclicLogic()` every 100 ms.

```pascal
PROGRAM MAIN
VAR
    Machine : Machine('Machine');
END_VAR

IF NOT Machine.Initialized THEN
    Machine.Initialize();
    RETURN;
END_IF

Machine.CyclicLogic();
```

## Machine Function Block

`Machine EXTENDS PackMLModule IMPLEMENTS I_Machine`

The top-level function block. It registers all sub-objects in `FB_Init` and overrides the PackML state methods.

**Registered sub-objects:**

| Object | Type | Purpose |
|--------|------|---------|
| `Events` | `TcEventClass` | Raises machine-level TwinCAT events |
| `EventReactor` | `EventReactorPackMLVisitor` | Reacts to events with PackML state transitions |
| `Machine_HMI` | `Machine_HMI` | HMI symbol and button handling |
| `SyncedSquareWaves` | `SyncedSquareWaves` | Demo cyclic component |
| `RecipeFile` | `JsonFile<10000>` | Loads `ST_Recipe` from `C:\Data\Recipe.json` |

## Initialization Sequence

`Initializing()` runs as a sequenced state machine before the PackML state machine starts:

| Step | Action |
|------|--------|
| 0 | Configure Production and Manual unit modes via `FB_PMLUnitModeConfig` |
| 10 | Call `RecipeFile.LoadFile()` |
| 20 | Wait for load to finish, then call `RecipeFile.ReadSymbol()` |
| 30 | Set `ModuleInitialized := TRUE` |

## PackML States

`Machine` extends `PackMLModule` and overrides the transition methods:

| State method | Behaviour |
|--------------|-----------|
| `Clearing` | Calls `ResetComponents()` and `ResetModules()`, then completes |
| `Starting` | Completes immediately |
| `Execute` | Empty — add production logic here |
| `Stopping` | Completes immediately |
| `Aborting` | Completes immediately |
| `Stopped`, `Aborted` | No override |

**Unit modes configured:** Production, Manual. Both allow mode changes from Stopped and Aborted states.

## Cyclic Logic

`OnCyclicCall()` runs every scan inside `CyclicLogic()`:

- **Manual mode** — `HmiEnableDisableAllVisitor` enables HMI on every node in the tree, allowing individual component control from the HMI.
- **All other modes** — HMI is disabled on the tree; only the top-level `Machine_HMI` is re-enabled.
- `SetAllOverrideVisitor` propagates `_Override` to every component in the hierarchy each scan.
- Raises `E_Machine.EStop_Pressed` when the E-stop is not OK.

## HMI

`Machine_HMI EXTENDS PackMLStatemachine_HMI`

Handles the three standard operator buttons and writes the HMI struct to the TwinCAT HMI symbol server.

| Button | Permissive | Action |
|--------|-----------|--------|
| Start | Current state = Stopped | `ChangeState(Start)` |
| Stop | Current state ≠ Aborted | `ChangeState(Stop)` (or `Abort` when already Stopped) |
| Reset | Current state = Aborted | `ChangeState(Clear)` (or `Reset` otherwise) |

The `ST_Machine_HMI` struct is published as a TwinCAT HMI symbol (accessible to Administrators and Admin group):

```pascal
TYPE ST_Machine_HMI EXTENDS ST_HmiFunction :
    STRUCT
        Override : LREAL := 100.0;
    END_STRUCT
END_TYPE
```

## Interface

`I_Machine EXTENDS I_ContainStatemachine`

| Member | Type | Direction | Purpose |
|--------|------|-----------|---------|
| `Override` | `LREAL` | set | Speed / override percentage applied to all components |
| `Recipe` | `REFERENCE TO ST_Recipe` | get | Active recipe values |

## Recipe

`ST_Recipe` is loaded from `C:\Data\Recipe.json` at startup and re-read into the symbol `Recipe` via the ADS symbol interface.

```
ST_Recipe
└── Axis : ST_AxisRecipe
    ├── Velocity     : LREAL := 100
    ├── Acceleration : LREAL := 1000
    ├── Deceleration : LREAL := 1000
    └── Jerk         : LREAL := 1000
```

## Extending the Template

To build a real application from this template:

1. **Add components and sub-modules** — declare them inside `Machine` and register them in `FB_Init` with `RegisterWithParent(THIS^)`.
2. **Implement state logic** — fill in the `Starting`, `Execute`, `Stopping`, and `Clearing` methods with the production sequence.
3. **Extend the recipe** — add fields to `ST_Recipe` and update `C:\Data\Recipe.json` accordingly.
4. **Add events** — define event enums in the `.tmc` event class file and raise them via `Events.RaiseEvent(...)`.
5. **Extend the HMI** — add fields to `ST_Machine_HMI` and handle them in `Machine_HMI.CyclicLogic()`.
