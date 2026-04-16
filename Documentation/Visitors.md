# Visitors

The visitor pattern allows operations to be applied across the entire component/module tree without modifying the visited objects. A visitor is passed to `Components.Accept()` or `Modules.Accept()` on any module, and it receives a callback for each item in the collection.

## How It Works

```
Module.Components.Accept(MyVisitor)
  → calls MyVisitor.VisitComponent(Component1)
  → calls MyVisitor.VisitComponent(Component2)
  → ...
```

To traverse sub-modules recursively, call `Accept` on `Module.Modules` as well, and within `VisitModule()` call `Accept` again on that module's collections.

---

## Available Visitors

### Enable / Disable

| Visitor | Purpose |
|---------|---------|
| `EnableAllComponentsVisitor` | Calls `Enable()` on every `I_Enablable` component in the collection |
| `DisableAllComponentsVisitor` | Calls `Disable()` on every `I_Enablable` component |
| `CheckComponentsEnabledVisitor` | Returns TRUE if all components are enabled |

```pascal
VAR
    EnableVisitor : EnableAllComponentsVisitor;
END_VAR

Module.Components.Accept(EnableVisitor);
```

### Reset

| Visitor | Purpose |
|---------|---------|
| `ResetAllVisitor` | Resets both components and modules |
| `ResetComponentsVisitor` | Resets components only |
| `ResetModulesVisitor` | Resets modules only |

```pascal
VAR
    ResetVisitor : ResetAllVisitor;
END_VAR

Module.Components.Accept(ResetVisitor);
Module.Modules.Accept(ResetVisitor);
```

### Stop

| Visitor | Purpose |
|---------|---------|
| `StopAllComponentsVisitor` | Calls `Stop()` on every stoppable component |

### Mode & State Change

| Visitor | Purpose |
|---------|---------|
| `ChangeModeOnAllSubModulesVisitor` | Propagates a mode change to all sub-modules |
| `ChangeStateOnAllSubModulesVisitor` | Propagates a state change to all sub-modules |
| `ChangePackMLStateOnAllSubModulesVisitor` | PackML-specific state propagation |

```pascal
VAR
    ModeVisitor : ChangeModeOnAllSubModulesVisitor(RequestedMode := E_Mode.Auto);
END_VAR

Machine.Modules.Accept(ModeVisitor);
```

### Events

| Visitor | Purpose |
|---------|---------|
| `EventReactorVisitor` | Triggers state transitions in response to active events |
| `EventReactorPackMLVisitor` | PackML variant of event-driven state changes |
| `GetActiveEventSeverityVisitor` | Collects the highest active event severity across all components |

### Force

| Visitor | Purpose |
|---------|---------|
| `ForceVisitor` | Sets a force state on all forcible digital components |
| `ReleaseAllForceVisitor` | Releases all active forces back to `Unforced` |

### Safety

| Visitor | Purpose |
|---------|---------|
| `SafetyOkVisitor` | Returns TRUE if all safety components report OK |
| `SetSafetySimulationVisitor` | Enables or disables simulation mode on safety components |

### HMI

| Visitor | Purpose |
|---------|---------|
| `GetSystemTreeJsonHmiVisitor` | Builds a JSON representation of the component/module tree for HMI |
| `HmiEnableDisableAllVisitor` | Enables or disables all HMI-exposed components |

### Task Results

| Visitor | Purpose |
|---------|---------|
| `GetComponentsTaskResultVisitor` | Aggregates `I_TaskResult` status across all components |

---

## Example — Collecting Component Names (from Component_TEST)

```pascal
VAR
    StringCollection : StringCollection<100>;
    Visitor          : ComponentVisitor_Mockup(StringCollection);
    Component        : Component_Mockup('MySensor');
END_VAR

Component.Accept(Visitor);
// StringCollection.Entries -> 1  (visitor was called once)

// Or across a module's whole collection
Module.Components.Accept(Visitor);
// StringCollection.Entries -> number of components in module
```

## Implementing a Custom Visitor

```pascal
FUNCTION_BLOCK MyVisitor IMPLEMENTS I_ComponentVisitor
METHOD VisitComponent
VAR_INPUT
    Component : I_Component;
END_VAR
    // operate on each component
    IF Component.Name = 'TargetSensor' THEN
        // ...
    END_IF
END_METHOD
```
