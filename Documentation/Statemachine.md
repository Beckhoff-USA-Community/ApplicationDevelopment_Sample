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

Actual := ExecuteState.IsExecuting;
// -> TRUE

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
