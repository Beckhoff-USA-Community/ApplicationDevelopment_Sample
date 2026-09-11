# Interfaces

Standalone interface definitions used throughout the framework. All functional interfaces extend `I_Base` to enable runtime interface casting via `__QueryInterface()`.

This catalogue covers the cross-cutting interfaces only. Domain interfaces are documented with their subsystem — see [Motion](Motion.md) for the axis interfaces, [Component](Component.md) for `I_Component` and `I_Cyclic`, and [Module](Module.md) for `I_Module`.

`__QueryInterface()` is also how the framework models **optional capabilities**: rather than pushing a feature into a subtype, the feature gets its own small interface and consumers query for it. `I_AxisGear` is the worked example — see [Gearing is a capability, not a subtype](Motion.md#gearing-is-a-capability-not-a-subtype).

---

## I_Base

Foundation interface for all functional interfaces in the framework. Extending `I_Base` enables `__QueryInterface()` casting, which visitors use to discover optional capabilities on visited objects.

---

## I_Command

| Member | Type | Description |
|--------|------|-------------|
| `Execute()` | Method | Execute the encapsulated command |

---

## I_Disposable

| Member | Type | Description |
|--------|------|-------------|
| `Dispose()` | Method | Release resources for a dynamically created function block |

---

## I_Enablable

| Member | Type | Description |
|--------|------|-------------|
| `Enabled` | `BOOL` (Get) | `TRUE` when the function is active |
| `Enable()` | Method | Activate the function |
| `Disable()` | Method | Deactivate the function |

---

## I_Initializable

| Member | Type | Description |
|--------|------|-------------|
| `Initialized` | `BOOL` (Get/Set) | `TRUE` after initialization completes |
| `Initialize()` | Method | Run the initialization sequence |

`Initialize()` is expected to be idempotent and is normally driven by the owning `Module`'s `Initializer` — see [Module](Module.md). A component that can also run unregistered should drive itself from `CyclicLogic()` **only** when it has no parent, so the call keeps a single owner; [`Mc3Axis`](Motion.md#who-drives-initialize) does exactly that.

---

## I_Override

| Member | Type | Description |
|--------|------|-------------|
| `Override` | (Set) | Apply an override to the function |

---

## I_Resettable

| Member | Type | Description |
|--------|------|-------------|
| `Reset()` | Method | Reset the function block to its initial state |

---

## I_Stoppable

| Member | Type | Description |
|--------|------|-------------|
| `Stopped` | `BOOL` (Get) | `TRUE` when the function has stopped |
| `Stop()` | Method | Command the function to stop |

---

## I_TaskResult

Common error/busy reporting interface implemented by asynchronous function blocks.

| Member | Type | Description |
|--------|------|-------------|
| `Error` | `BOOL` (Get) | `TRUE` when the function block is in an error state |
| `Busy` | `BOOL` (Get) | `TRUE` while an operation is in progress |
| `ErrorId` | `UDINT` (Get) | Numeric error code |
| `ErrorDescription` | `STRING` (Get) | Human-readable error description |
