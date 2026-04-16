# Collections & Buffers

---

## AnyBuffer

A generic double-ended queue (deque) that works with any data type. Can operate as a FIFO, LIFO, ring buffer, or random-access list. Data and its positional metadata are stored together so the buffer survives PLC restarts with retain variables.

Implements: `I_AnyBuffer`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `Clear()` | Method | Zeros all data and resets metadata |
| `PushLeft(Value)` | Method → `BOOL` | Inserts at the front; returns FALSE if full |
| `PushRight(Value)` | Method → `BOOL` | Inserts at the back; returns FALSE if full |
| `PopLeft()` | Method → `BOOL` | Removes from the front; returns FALSE if empty |
| `PopRight()` | Method → `BOOL` | Removes from the back; returns FALSE if empty |
| `PeekAtIndex(Index)` | Method → value | Non-destructive read at position |
| `InsertAtIndex(Index, Value)` | Method → `BOOL` | Inserts at position, shifts rest right |
| `DeleteAtIndex(Index)` | Method → `BOOL` | Removes at position, shifts rest left |
| `ReplaceAtIndex(Index, Value)` | Method → `BOOL` | Overwrites value at position |

### Specialised Variants

| FB | Behaviour |
|----|-----------|
| `FifoBuffer` | Push right, pop left |
| `LifoBuffer` | Push right, pop right (stack) |
| `RingBuffer` | Fixed-size circular overwrite on push |
| `ListBuffer` | Linked-list style insertion/deletion |

### Example

```pascal
// From AnyBuffer_TEST — FIFO usage
VAR
    Buffer : FifoBuffer;
    Value  : INT;
END_VAR

Buffer.PushRight(10);
Buffer.PushRight(20);
Buffer.PushRight(30);

Buffer.PopLeft();
Value := Buffer.PeekAtIndex(0);
// -> 20  (10 was removed)
```

---

## ComponentCollection\<MaxSize\>

Typed collection of `I_Component` references with name-based lookup and visitor dispatch. Used internally by modules but also useful for custom aggregations.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `AddComponent(Component)` | Method → `RegistrationResult` | Adds a component; fails if full or duplicate name |
| `RemoveComponent(Component)` | Method → `RegistrationResult` | Removes a component |
| `GetComponentByName(Name)` | Method → `I_Component` | Returns component or null |
| `Accept(Visitor)` | Method | Dispatches `VisitComponent()` for each entry |
| `Count` | `UDINT` (Get) | Number of registered components |

### Example

```pascal
// From Component_TEST — registration and lookup
VAR
    Collection : ComponentCollection<50>;
    Sensor     : Component_Mockup('PressureSensor');
END_VAR

Collection.AddComponent(Sensor);
Result := Collection.GetComponentByName('PressureSensor').Name;
// -> 'PressureSensor'
```

---

## Collection\<MaxSize\>

Lower-level generic collection of `I_Base` items with enumeration support. `ComponentCollection` and `ModuleCollection` are built on top of this.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `AddItem(Item)` | Method | Appends an item |
| `GetItem(Index)` | Method → `I_Base` | Returns item at zero-based index |
| `Clear()` | Method | Removes all items |
| `Count` | `UDINT` (Get) | Number of items |
| `MoveFirst()` / `MoveNext()` | Methods | Enumerator support |
| `Current` | `I_Base` (Get) | Current enumerator item |
