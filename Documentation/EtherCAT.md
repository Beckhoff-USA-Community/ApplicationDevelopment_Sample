# EtherCAT Components

## EtherCatMaster\<FRAMES, SYNC_UNITS\>

Cyclic component that manages an EtherCAT master — reads slave topology, monitors slave and master states, and exposes per-slave `EtherCatIoDevice` instances. Requires hardware mapping of the `EcMaster` and `FrmXWcState` variables to the TwinCAT EtherCAT master task.

Extends: `CyclicComponent`  
Implements: `I_EtherCatMasterDiagnostic`, `I_Initializable`, `I_TaskResult`, `I_Simulatable`

### Generic Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `FRAMES` | 1 | Number of EtherCAT frames; must match the master configuration |
| `SYNC_UNITS` | 2 | Number of DC sync units |

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, EcStartupCycles)` | Constructor | `EcStartupCycles`: number of scans to wait before considering slaves ready |
| `Initialized` | `BOOL` (Get/Set) | TRUE after `Initialize()` completes successfully |
| `Simulated` | `BOOL` (Get/Set) | If TRUE, skips hardware checks and initialises in simulation |
| `Busy` | `BOOL` (Get) | TRUE while a slave diagnostic sequence is running |
| `Error` | `BOOL` (Get) | TRUE if master or any slave is in a fault state |
| `ErrorId` | `UDINT` (Get) | Master state word or faulted slave index |
| `ConfiguredSlaveCount` | `UINT` (Get) | Number of configured slaves |
| `SlaveCount` | `UINT` (Get) | Number of currently active slaves |
| `LocalAmsNetId` | `T_AmsNetId` (Get) | AMS Net ID of the local runtime |
| `MasterAmsNetId` | `T_AmsNetId` (Get) | AMS Net ID of the EtherCAT master |
| `Initialize()` | Method | Reads slave config, frame count, and initialises sync units |
| `Reset()` | Method | Clears error state and re-enables diagnostics |
| `CyclicLogic()` | Method | Must be called each scan |
| `GetIoDeviceByAddr(Addr)` | Method → `I_EcIoDevice` | Returns the `EtherCatIoDevice` for a given slave address |
| `GetIoDeviceByName(Name)` | Method → `I_EcIoDevice` | Returns the `EtherCatIoDevice` for a given slave name |
| `RegisterEventProvider(Provider)` | Method | Plugs in a TcEvent publisher for master/slave diagnostics |

### Example

```pascal
VAR
    EcMaster : EtherCatMaster<1, 2>('EtherCAT Master', EcStartupCycles := 3);
END_VAR

// In MAIN cyclic code
EcMaster.CyclicLogic();

IF EcMaster.Error THEN
    // Handle master or slave fault
END_IF

// Access a specific slave's I/O device
Device := EcMaster.GetIoDeviceByName('EL2008');
IF Device.OP THEN
    // Slave is in operational state, I/O data is valid
END_IF
```

---

## EtherCatIoDevice

Wraps a single EtherCAT slave. Created and managed internally by `EtherCatMaster` — one instance per configured slave. Exposes the slave's ESM state, link state, and CoE access.

Extends: `CyclicComponent`  
Implements: `I_EcIoDevice`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `Name` | `STRING` (Get/Set) | Slave name from the EtherCAT configuration |
| `Configuration` | `REFERENCE TO ST_EcSlaveConfigData` (Get/Set) | Slave configuration data (address, mailbox sizes, etc.) |
| `State` | `REFERENCE TO ST_EcSlaveState` (Get/Set) | Live slave state from the master diagnostic |
| `AmsNetId` | `AMSNETID` (Set) | Master AMS Net ID, set by `EtherCatMaster` during init |
| `SyncUnit` | `I_SyncUnitTask` (Set) | DC sync unit this slave belongs to |
| `CoE` | `I_CoeDevice` (Get) | CoE access; returns `NullCoeDevice` if slave has no mailbox |
| `OP` | `BOOL` (Get) | TRUE when slave is in Operational state |
| `SafeOp` | `BOOL` (Get) | TRUE when slave is in Safe-OP |
| `PreOp` | `BOOL` (Get) | TRUE when slave is in Pre-OP |
| `Init` | `BOOL` (Get) | TRUE when slave is in Init state |
| `Bootstrap` | `BOOL` (Get) | TRUE when slave is in Bootstrap state |
| `Disabled` | `BOOL` (Get) | TRUE when slave is disabled |
| `DeviceError` | `BOOL` (Get) | TRUE when device state error bit is set |
| `IoDataValid` | `BOOL` (Get) | TRUE when sync unit I/O data is valid (or no sync unit assigned) |
| `InvalidVPRS` | `BOOL` (Get) | TRUE when vendor/product/revision/serial mismatch |
| `CyclicLogic()` | Method | Services the internal `CoeDevice`; called by `EtherCatMaster` |
