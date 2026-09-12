# CoE (CAN over EtherCAT)

## CoeDevice

Cyclic component that wraps `FB_EcCoESdoReadEx` and `FB_EcCoESdoWriteEx` to provide SDO read/write access to an EtherCAT slave's CoE object dictionary. Used by `EtherCatIoDevice` and `SafetyModule` to read device parameters and safety address information at startup.

Extends: `CyclicComponent`  
Implements: `I_CoeDevice`, `I_CoeTransfer`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name)` | Constructor | Standard component name |
| `AdsAddr` | `REFERENCE TO AMSADDR` (Get/Set) | AMS address of the target slave (net ID + port = slave address) |
| `Scheduler` | `I_CoeTransferScheduler` (Get/Set) | Optional. When set, `Read`/`Write` register the device and the scheduler calls `CyclicLogic` until the transfer completes (`EtherCatMaster` does this for its io devices) |
| `Busy` | `BOOL` (Get) | TRUE while a read or write SDO is in progress |
| `Error` | `BOOL` (Get) | TRUE if the last started operation faulted |
| `ErrorId` | `UDINT` (Get) | EtherCAT/ADS error code; `16#70B` if `AdsAddr` was not set |
| `Read(Index, SubIndex, pDstBuf, BufLen)` | Method | Initiates an SDO read; ignored if busy |
| `Write(Index, SubIndex, pDstBuf, BufLen)` | Method | Initiates an SDO write from `pDstBuf`; ignored if busy |
| `CyclicLogic()` | Method | Services the pending SDO; call each scan while `Busy` unless a `Scheduler` is set |

### Notes

- `SubIndex = 0` automatically sets `bCompleteAccess = TRUE` (reads all sub-objects).
- Only one operation can be active at a time; calls while busy return immediately.
- `Error`/`ErrorId` describe the last started transfer only; a stale error of the other function block does not leak through.
- Requires a valid `AdsAddr` reference before use; `Read`/`Write` without one set `Error` with `ErrorId = 16#70B`.

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

---

## I_CoeTransfer / I_CoeTransferScheduler

Scheduling contract that lets an owner of many `CoeDevice` instances service only the transfers in flight instead of calling every device each cycle.

| Interface | Members | Description |
|-----------|---------|-------------|
| `I_CoeTransfer` | extends `I_Cyclic`, `I_TaskResult` | A transfer the scheduler drives with `CyclicLogic()` while `Busy` |
| `I_CoeTransferScheduler` | `Register(Transfer : I_CoeTransfer) : BOOL` | Adds a transfer to the serviced set (idempotent); FALSE if the set is full |

`EtherCatMaster` implements `I_CoeTransferScheduler` and passes itself to every `EtherCatIoDevice`; a `CoeDevice` with a `Scheduler` registers itself in `Read`/`Write` and is dropped from the set when `Busy` falls.
