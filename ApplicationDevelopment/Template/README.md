# Template

Minimal, pre-wired TwinCAT application that ties the [`ApplicationBase`](../../ApplicationBase) framework into a working starting point. Provides the scaffold — `Machine` extending `PackMLModule`, HMI, recipe loading from JSON, and a TwinCAT event class — all wired and ready to extend.

## Running

1. Open [`ApplicationDevelopment.sln`](../ApplicationDevelopment.sln) in TwinCAT XAE.
2. Make sure `C:\Data\Recipe.json` exists on the target (the recipe loader reads it at startup).
3. Activate the `Template` PLC task, log in, and start the runtime.

`MAIN.TcPOU` instantiates `Machine('Machine')`, runs `Initialize()` until complete, then calls `CyclicLogic()` every 100 ms.

## Where to Extend

| Want to add … | Where |
|---------------|-------|
| New components or sub-modules | Declare in `Machine` and register in `FB_Init` with `RegisterWithParent(THIS^)` |
| Production logic | Fill in `Starting`, `Execute`, `Stopping`, `Clearing` on `Machine` |
| Recipe fields | Extend `ST_Recipe` and update `C:\Data\Recipe.json` |
| Events | Add enums to `Events_Template.tmc` and raise via `Events.RaiseEvent(...)` |
| HMI controls | Extend `ST_Machine_HMI` and handle in `Machine_HMI.CyclicLogic()` |

## Full Documentation

See [Documentation/Template.md](../../Documentation/Template.md) for the full walkthrough — entry point, PackML states, initialization sequence, HMI, and recipe structure.
