# CoE (CAN over EtherCAT)

## CoeDevice

Cyclic component that wraps `FB_EcCoESdoReadEx` to provide SDO read/write access to an EtherCAT slave's CoE object dictionary. Used by `EtherCatIoDevice` and `SafetyModule` to read device parameters and safety address information at startup.

Extends: `CyclicComponent`  
Implements: `I_CoeDevice`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name)` | Constructor | Standard component name |
| `AdsAddr` | `REFERENCE TO AMSADDR` (Get/Set) | AMS address of the target slave (net ID + port = slave address) |
| `Busy` | `BOOL` (Get) | TRUE while a read or write SDO is in progress |
| `Error` | `BOOL` (Get) | TRUE if the last operation faulted |
| `ErrorId` | `UDINT` (Get) | EtherCAT error code |
| `Read(Index, SubIndex, pDstBuf, BufLen)` | Method | Initiates an SDO read; ignored if busy |
| `Write(Index, SubIndex, pDstBuf, BufLen)` | Method | Initiates an SDO write; ignored if busy |
| `CyclicLogic()` | Method | Must be called each scan; services pending SDOs |

### Notes

- `SubIndex = 0` automatically sets `bCompleteAccess = TRUE` (reads all sub-objects).
- Only one operation can be active at a time; calls while busy return immediately.
- Requires a valid `AdsAddr` reference before use; returns without action if the reference is invalid.

### Example

```pascal
// Reading safety address info from a TwinSAFE PLC (from SafetyModule.Initializing)
VAR
    SafetyAddr : AMSADDR;
    CoE        : CoeDevice('SafetyCoE');
    FsoeAddr   : DWORD;
END_VAR

CoE.AdsAddr REF= SafetyAddr;
CoE.Read(16#F980, 1, ADR(FsoeAddr), SIZEOF(FsoeAddr));

// Call each scan until not busy
CoE.CyclicLogic();
IF NOT CoE.Busy AND NOT CoE.Error THEN
    // FsoeAddr now contains the FSoE address
END_IF
```

---

## NullCoeDevice

Null-object implementation of `I_CoeDevice`. Returns `Busy = FALSE`, `Error = FALSE`, and ignores all Read/Write calls. Automatically substituted by `EtherCatIoDevice` when a slave has no mailbox (no CoE support).

No configuration required — instantiate and use as a safe no-op placeholder.
