# Statemachine

A generic, index-based state machine that extends `CyclicComponent`. States self-register at construction time, the machine dispatches `Executing()` to the active state each scan, and transitions are triggered by calling `ChangeState()`.

## State

Abstract base for individual states. Each state implementation extends `State` and overrides `Executing()`.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(State, Statemachine)` | Constructor | Registers this state at index `State` in the given machine |
| `SequenceState` | `UDINT` | Step counter for internal sequencing within a state |
| `Executing()` | Abstract Method | Called every scan while this state is active; return TRUE to stay, FALSE to signal completion |
| `Inactive()` | Method | Called when not active; resets `SequenceState := 0` |

Implements: `I_State`

---

## Statemachine\<InitialState\>

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name)` | Constructor | Names the machine; `InitialState` generic sets the startup state index |
| `CurrentState` | `DINT` (Get) | Index of the currently active state |
| `States` | `I_StateList` (Get) | Access the state registry (add states manually if not using FB_Init auto-registration) |
| `Mode` | `I_Mode` (Get/Set) | Pluggable mode controller; defaults to `ModeControl` |
| `ChangeState(StateCommand)` | Method | Transitions to the state at index `StateCommand` |
| `CyclicLogic()` | Method | Calls `Executing()` on the active state and `Inactive()` on all others |
| `Accept(Visitor)` | Method | Visitor pattern support |

Extends: `CyclicComponent`  
Implements: `I_Statemachine`

---

## Example

```pascal
// From Statemachine_TEST — basic setup and state verification
VAR
    Statemachine : Statemachine<1>('Seal Station');  // starts in state index 1
    IdleState    : State_Mockup(0, Statemachine);    // index 0
    ExecuteState : State_Mockup(1, Statemachine);    // index 1
END_VAR

// States self-register on FB_Init — no manual AddStateAtIndex needed
Statemachine.CyclicLogic();
Actual := Statemachine.CurrentState;
// -> 1  (InitialState generic)

// Transition to state 0
Statemachine.ChangeState(0);
Actual := Statemachine.CurrentState;
// -> 0

// Manually add states (when not using constructor auto-register)
Statemachine.States.AddStateAtIndex(0, IdleState);
Statemachine.States.AddStateAtIndex(1, ExecuteState);
GetState := Statemachine.States.GetStateAtIndex(1);
// -> reference to ExecuteState
```

## PackML Statemachine

A pre-built statemachine conforming to the PackML V3 standard (`Tc2_PackML_V3`). It starts in the `Aborted` state and drives all standard PackML states and mode transitions.

### Setup

**Step 1 — Add `Tc2_PackML_V3` to your project.** Only V3 is supported.

**Step 2 — Declare the statemachine and implement `I_ContainStatemachine`:**

```pascal
FUNCTION_BLOCK VFFSMachine EXTENDS Module IMPLEMENTS I_ContainStatemachine
VAR
    _Statemachine       : PackMLStatemachine('VFFS Machine');
    Statemachine_HMI    : PackMLStatemachine_HMI('Statemachine HMI', _Statemachine);
END_VAR
```

**Step 3 — Configure modes in `Initializing()`:**

```pascal
METHOD Initializing
VAR_INST
    UnitModeConfig : FB_PMLUnitModeConfig;
    SequenceState  : UDINT;
END_VAR

CASE SequenceState OF
    0:
        UnitModeConfig(
            eMode                         := E_PMLProtectedUnitMode.Production,
            sName                         := 'Mode Production',
            bEnableUnitModeChangeStopped  := TRUE,
            bEnableUnitModeChangeAborted  := TRUE,
            (* set remaining bDisable* / bEnableUnitModeChange* flags as needed *)
            bError   =>,
            nErrorId =>);

        ModuleInitialized := TRUE;
END_CASE
```

**Step 4 — Instantiate and register the PackML states:**

```pascal
FUNCTION_BLOCK VFFSMachine EXTENDS Module IMPLEMENTS I_ContainStatemachine
VAR
    _Statemachine     : PackMLStatemachine('VFFS Machine');
    State_Aborted     : MachineState_Aborted(E_PMLState.Aborted, _Statemachine, THIS^);
    State_Aborting    : MachineState_Aborting(E_PMLState.Aborting, _Statemachine, THIS^);
    State_Clearing    : MachineState_Clearing(E_PMLState.Clearing, _Statemachine, THIS^);
    State_Stopping    : MachineState_Stopping(E_PMLState.Stopping, _Statemachine, THIS^);
    State_Stopped     : MachineState_Stopped(E_PMLState.Stopped, _Statemachine, THIS^);
    State_Resetting   : MachineState_Resetting(E_PMLState.Resetting, _Statemachine, THIS^);
    State_Idle        : MachineState_Idle(E_PMLState.Idle, _Statemachine, THIS^);
    State_Starting    : MachineState_Starting(E_PMLState.Starting, _Statemachine, THIS^);
    State_Execute     : MachineState_Execute(E_PMLState.Execute, _Statemachine, THIS^);
    Statemachine_HMI  : PackMLStatemachine_HMI('Statemachine HMI', _Statemachine);
END_VAR
```

**Step 5 — Register in `FB_Init`:**

```pascal
_Statemachine.RegisterWithParent(THIS^);
Statemachine_HMI.RegisterWithParent(THIS^);
```

**Step 6 — Access state and mode at runtime:**

```pascal
// Current PackML state
_Statemachine.CurrentState        // E_PMLState enum value

// Current mode
_Statemachine.Mode.CurrentMode    // E_PMLProtectedUnitMode enum value

// Command a transition
_Statemachine.ChangeState(E_PMLCommand.Start);
```

`E_PMLProtectedUnitMode` and `E_PMLState` from `Tc2_PackML_V3` are available throughout.

---

## Implementing a Custom State

```pascal
// Extend State and override Executing()
FUNCTION_BLOCK SealingState EXTENDS State
VAR
    Timer : TON;
END_VAR

METHOD Executing : BOOL
    Timer(IN := TRUE, PT := T#2S);
    IF Timer.Q THEN
        Statemachine.ChangeState(E_MyStates.Cooling);
        Executing := FALSE;
    ELSE
        Executing := TRUE;
    END_IF
END_METHOD
```
