# Module

A `Module` is a logical container that groups components and sub-modules representing a physical or functional section of a machine (e.g., a forming station, a conveyor, an entire machine). It coordinates initialization and cyclic execution of everything it contains.

The framework provides two concrete module types:
- **`EquipmentModule`** — represents a sub-section of a machine (e.g., a single axis station).
- **`MachineModule`** — the top-level container; holds equipment modules and top-level components.

## Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name)` | Constructor | Assigns the module name |
| `Name` | `STRING` (Get/Set) | Module identifier |
| `Initialized` | `BOOL` (Get/Set) | TRUE after `Initialize()` completes |
| `Components` | `I_ComponentCollection` (Get) | Collection of registered components |
| `Modules` | `I_ModuleCollection` (Get) | Collection of registered sub-modules |
| `RegisterComponent(Component)` | Method | Adds a component to the component collection |
| `RegisterCyclic(Cyclic)` | Method | Registers an `I_Cyclic` for dispatch each scan |
| `RegisterModule(Module)` | Method | Adds a sub-module |
| `RegisterInitialize(Item)` | Method | Adds an `I_Initializable` to the init sequence |
| `DeregisterComponent(Component)` | Method | Removes a component |
| `DeregisterModule(Module)` | Method | Removes a sub-module |
| `Initialize()` | Method | Runs the initialization sequence |
| `CyclicLogic()` | Method | Dispatches cyclic logic to all registered items |
| `GetComponentByName(Name)` | Method | Looks up a component by name |
| `GetModuleByName(Name)` | Method | Looks up a sub-module by name |
| `OnCyclicCall()` | Abstract Method | Custom scan-level logic (override in subclass) |
| `Initializing()` | Abstract Method | Custom init logic; must set `ModuleInitialized := TRUE` |

Implements: `I_Module`, `I_Cyclic`, `I_Initializable`

## Example

```pascal
// From Module_TEST — cyclic dispatch reaches registered component
VAR
    CyclicComponent : CyclicComponent_Mockup('Pump');
    Module          : Module_Mockup('Station');
END_VAR

Module.RegisterCyclic(CyclicComponent);
Module.CyclicLogic();
Actual := CyclicComponent.WasCalled;
// -> TRUE

// From Module_TEST — component lookup by name
VAR
    Component : CyclicComponent_Mockup('Sensor');
    Module    : Module_Mockup('Station');
END_VAR

Module.RegisterComponent(Component);
Module.CyclicLogic();
Actual := Module.GetComponentByName('Sensor').Name;
// -> 'Sensor'

// From Module_TEST — nested module hierarchy (MachineModule containing EquipmentModule)
VAR
    C1, C2 : CyclicComponent_Mockup('C1');
    Em     : EquipmentModule_Mockup('FormStation', C1, C2);
    C3, C4 : CyclicComponent_Mockup('C3');
    Machine : MachineModule_Mockup('VFFS', Em, C3, C4);
END_VAR

Machine.CyclicLogic();
ActualModule := Machine.GetModuleByName('FormStation');
// -> reference to Em

// From Module_TEST — initialization sequence
VAR
    InitComp : InitializableComponent_Mockup('SeqInit');
    Module   : Module_Mockup('Station');
END_VAR

InitComp.Initialized := TRUE;
Module.RegisterInitialize(InitComp);
Module.Initialize();
Actual := Module.Initialized;
// -> TRUE
```
