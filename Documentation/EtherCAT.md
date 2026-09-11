# EtherCAT Components

## EtherCatMaster\<FRAMES, SYNC_UNITS\>

Cyclic component that manages an EtherCAT master — reads the slave configuration and topology once, monitors master and slave states on every change, and exposes per-slave `EtherCatIoDevice` instances. Requires hardware mapping of the `EcMaster` and `FrmXWcState` variables to the TwinCAT EtherCAT master task.

Built to scale: the per-cycle cost is either constant or bounded by `SLAVES_PER_CYCLE`, every ADS read is sized to the configured slave count, slave lookups by address go through an O(1) map, and only CoE transfers in flight are serviced. Verified target: 2500 slaves on a 10 ms task.

Extends: `CyclicComponent`  
Implements: `I_EtherCatMasterDiagnostic`, `I_Initializable`, `I_TaskResult`, `I_Simulatable`, `I_CoeTransferScheduler`

### Generic Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `FRAMES` | 1 | Number of EtherCAT frames; must match the master configuration |
| `SYNC_UNITS` | 2 | Number of DC sync units |

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, EcStartupCycles)` | Constructor | `EcStartupCycles`: number of scans to wait before wiring the slaves |
| `Initialized` | `BOOL` (Get/Set) | TRUE after `Initialize()` completes successfully |
| `Simulated` | `BOOL` (Get/Set) | If TRUE, skips hardware checks and initialises in simulation |
| `Busy` | `BOOL` (Get) | TRUE while a diagnostic sequence runs. A pass over n slaves takes up to n / `SLAVES_PER_CYCLE` cycles, so `Error` of a change lands that many cycles after the trigger |
| `Error` | `BOOL` (Get) | TRUE if the master, a slave, an ADS read or the sequence itself faulted; latched until `Reset()` |
| `ErrorSource` | `E_EcErrorSource` (Get) | Origin of `ErrorId`: `None`, `Master`, `Slave`, `Ads`, `Sequence` |
| `ErrorId` | `UDINT` (Get) | Raw code of `ErrorSource`: `DevState` word (`Master`), configured slave index (`Slave`), function block error id or `16#745` after the watchdog (`Ads`), step number or offending count (`Sequence`) |
| `ConfiguredSlaveCount` | `UINT` (Get) | Number of configured slaves |
| `SlaveCount` | `UINT` (Get) | Number of currently active slaves |
| `LocalAmsNetId` | `T_AmsNetId` (Get) | AMS Net ID of the local runtime |
| `MasterAmsNetId` | `T_AmsNetId` (Get) | AMS Net ID of the EtherCAT master |
| `Initialize()` | Method | Reads slave configuration, topology and frame count, initialises the sync units, wires the io devices |
| `Reset()` | Method | Clears the error state and re-triggers the diagnostic |
| `CyclicLogic()` | Method | Must be called each scan |
| `GetIoDeviceByAddr(Addr)` | Method → `I_EcIoDevice` | O(1) lookup; returns the null device and logs a message for unknown addresses |
| `GetIoDeviceByName(Name)` | Method → `I_EcIoDevice` | Linear search; call once and keep the result. Returns the null device and logs a message on a miss |
| `GetSlaveIndexByAddr(Addr)` | Method → `DINT` | Configured slave index (0-based) for an EtherCAT address, -1 if unknown. O(1) for addresses 1001 .. 1001 + `MAX_EC_SLAVES`, linear search for user-assigned addresses outside that range |
| `RegisterEventProvider(Provider)` | Method | Plugs in a TcEvent publisher for master/slave diagnostics |
| `Register(Transfer)` | Method → `BOOL` | `I_CoeTransferScheduler`; called by `CoeDevice`, not by applications |

### Parameters (`EtherCatParameter`)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `MAX_EC_SLAVES` | 49 | Upper index of every per-slave array; holds `MAX_EC_SLAVES + 1` slaves. `Initialize()` fails with `ErrorSource = Sequence` if more slaves are configured |
| `SLAVES_PER_CYCLE` | 64 | Slaves handled per cycle by the multi-cycle passes (classification, evaluation, sync unit resolution, io device wiring) |
| `ADS_WAIT_CYCLES` | 1000 | Cycles a diagnostic ADS read may stay busy before the sequence aborts with `ErrorSource = Ads`, `ErrorId = 16#745` (10 s on a 10 ms task) |
| `INIT_WAIT_CYCLES` | 60000 | Same guard for the one-off configuration and topology reads in `Initialize()`. Generous because TwinCAT serves ADS reads above 128 KiB from a task without process data on the master very slowly (about 3 min for 2500 slaves) |

### Sequences

**Initialize** — local AMS Net ID → configured slaves (read sized to `CfgSlaveCount`) → topology (once; supplies the hot connect flags) → frame count and `FrmXWcState` mapping → sync units → after `EcStartupCycles`: address map, then io device wiring at `SLAVES_PER_CYCLE` per cycle. Every wait step is guarded by a cycle watchdog; a failure is logged through `Trace` and latched with `ErrorSource`/`ErrorId`.

**Diagnostic** — triggered by a change of `ChangeCount`, of `SlaveCount` while it differs from `CfgSlaveCount`, or of the frame working counter state:

1. Read all slave states (sized read).
2. Classify at `SLAVES_PER_CYCLE` per cycle until the first slave that is not OP with a clean link. A clean network ends here — no topology read, no evaluation.
3. Otherwise read the topology (hot connect flags) and evaluate from that first slave on. Members of hot connect groups are skipped, disabled slaves raise `SlaveDisabled`, the first real fault raises `SlaveError` with the decoded state and latches `Error` with `ErrorSource = Slave`, `ErrorId = index`.
4. Once, after every sync unit has resolved its slave list: assign each slave its sync unit (drives `IoDataValid`).

The master `DevState` is checked every cycle; one `MasterError` event is raised per state change while slaves are missing.

### Example

```pascal
VAR
    EcMaster : EtherCatMaster<1, 2>('EtherCAT Master', EcStartupCycles := 3);
    Device   : I_EcIoDevice;
END_VAR

// In MAIN cyclic code
EcMaster.CyclicLogic();

IF EcMaster.Error THEN
    CASE EcMaster.ErrorSource OF
        E_EcErrorSource.Slave: ; // EcMaster.ErrorId = configured index of the faulted slave
        E_EcErrorSource.Master: ; // EcMaster.ErrorId = DevState word
        E_EcErrorSource.Ads: ;    // EcMaster.ErrorId = ADS error id of the failed read
    END_CASE
END_IF

// Resolve a slave once (after Initialized) and keep the reference
IF EcMaster.Initialized AND Device = 0 THEN
    Device := EcMaster.GetIoDeviceByName('EL2008');
END_IF

IF Device <> 0 AND_THEN Device.OP THEN
    // Slave is in operational state, I/O data is valid
END_IF
```

### Changes in 2.1.0

- The per-slave `CyclicLogic()` loop is gone; `EtherCatMaster` implements `I_CoeTransferScheduler` and services only CoE transfers in flight.
- Address-to-index map (`GetSlaveIndexByAddr`) replaces the linear scans of the sync unit assignment and `GetIoDeviceByAddr`; the sync unit name resolution is O(n) instead of O(n²).
- All O(n) passes run at `SLAVES_PER_CYCLE` slaves per cycle; ADS reads are sized to the configured count and the topology is only re-read when a slave needs a diagnostic.
- `ErrorSource` tells what `ErrorId` means; the placeholder value 999 is gone, ADS errors keep the function block error id, and every wait step has a watchdog (`ADS_WAIT_CYCLES`, `INIT_WAIT_CYCLES`).
- Fixed: the master TcEvent was raised every cycle instead of once per `DevState` change; `DeviceError`, `Disabled`, `InvalidVPRS` and `InitCmdError` of `EtherCatIoDevice` were always FALSE; `CoeDevice.Write` used the SDO read function block; the sync unit re-read and re-matched its slaves on every working counter recovery and logged an error each pass for an empty sync unit; `Reset()` did not clear the frame state; init trace texts said "mapped" for "not mapped"; lookups by name or address now log a miss instead of silently returning the null device.

---

## EtherCatIoDevice

Wraps a single EtherCAT slave. Created and managed internally by `EtherCatMaster` — one instance per configured slave. Exposes the slave's ESM state, diagnostic flags, link state, sync unit validity and CoE access.

Extends: `CyclicComponent`  
Implements: `I_EcIoDevice`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `Name` | `STRING` (Get/Set) | Slave name from the EtherCAT configuration |
| `Configuration` | `REFERENCE TO ST_EcSlaveConfigData` (Get/Set) | Slave configuration data (address, mailbox sizes, etc.); setting it addresses the CoE mailbox |
| `State` | `REFERENCE TO ST_EcSlaveState` (Get/Set) | Live slave state from the master diagnostic |
| `AmsNetId` | `AMSNETID` (Set) | Master AMS Net ID, set by `EtherCatMaster` during init |
| `SyncUnit` | `I_SyncUnitTask` (Set) | DC sync unit this slave belongs to |
| `CoeScheduler` | `I_CoeTransferScheduler` (Set) | Scheduler that services this slave's CoE transfers; set by `EtherCatMaster` |
| `CoE` | `I_CoeDevice` (Get) | CoE access; returns `NullCoeDevice` if the slave has no mailbox or no configuration yet |
| `OP` / `SafeOp` / `PreOp` / `Init` / `Bootstrap` | `BOOL` (Get) | ESM state (`deviceState` bits 0..3) |
| `DeviceError` | `BOOL` (Get) | State machine error in the slave (`deviceState` bit 4) |
| `InvalidVPRS` | `BOOL` (Get) | Vendor id / product code / revision / serial mismatch (bit 5) |
| `InitCmdError` | `BOOL` (Get) | Error while sending the init commands (bit 6) |
| `Disabled` | `BOOL` (Get) | Slave is disabled in the configuration (bit 7) |
| `IoDataValid` | `BOOL` (Get) | TRUE when the sync unit I/O data is valid (or no sync unit assigned) |
| `CyclicLogic()` | Method | Services the internal `CoeDevice` only when no `CoeScheduler` is attached |

### Example

```pascal
VAR
    Device   : I_EcIoDevice;
    Identity : ST_TerminalProductInfo;
END_VAR

Device := EcMaster.GetIoDeviceByAddr(1003);

IF Device.OP AND NOT Device.CoE.Busy THEN
    Device.CoE.Read(16#1018, 0, ADR(Identity), SIZEOF(Identity)); // serviced by the master until done
END_IF
```
