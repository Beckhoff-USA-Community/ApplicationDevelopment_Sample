# Component & CyclicComponent

## Component

Abstract base class for all building blocks in the framework. Every reusable unit of I/O, control logic, or communication extends `Component`. It provides a name, visitor acceptance, and parent registration.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name)` | Constructor | Assigns the component name |
| `Name` | `STRING` (Get/Set) | Identifier used in collections and HMI tree |
| `Accept(Visitor)` | Method | Accepts an `I_ComponentVisitor` for tree traversal |
| `RegisterWithParent(Parent)` | Method | Adds itself to a parent module's component collection |
| `DeregisterWithParent(Parent)` | Method | Removes itself from a parent module's component collection |

Implements: `I_Component`

### Example

```pascal
// From Component_TEST — verifying name is assigned on construction
VAR
    Component : Component_Mockup('MyComponent');
END_VAR

Expected := 'MyComponent';
Actual   := Component.Name;
// -> passes

// Verify registration in a parent module
VAR
    Component : Component_Mockup('Sensor');
    Module    : EquipmentModule_Mockup('Station', Component, Component);
END_VAR

Expected := 'Sensor';
Actual   := Module.GetComponentByName('Sensor').Name;
// -> 'Sensor'

// Deregister removes it from the module
Component.DeregisterWithParent(Module);
Actual := Module.GetComponentByName('Sensor');
// -> 0 (null)
```

---

## CyclicComponent

Extends `Component` with cyclic execution support. Use this base when the component needs to run logic every PLC scan. Subclasses must implement the abstract `CyclicLogic()` method.

### Interface

Inherits all members of `Component`, plus:

| Member | Type | Description |
|--------|------|-------------|
| `CyclicLogic()` | Abstract Method | Called every scan by the parent module's `CyclicRunner` |

Implements: `I_Component`, `I_Cyclic`

When instantiated inside an `EquipmentModule_Mockup` or any module, the component is automatically registered for both component lookup and cyclic dispatch.

### Example

```pascal
// From CyclicComponent_TEST — verifying cyclic dispatch
VAR
    Component : CyclicComponent_Mockup('Pump');
    Module    : EquipmentModule_Mockup('Station', Component, Component);
END_VAR

Module.CyclicLogic();
Actual := Component.WasCalled;
// -> TRUE  (CyclicLogic was dispatched by the module)
```
