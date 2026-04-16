# Serial Communication Components

All serial components extend `CyclicComponent` and depend on an `I_SerialLineControl` instance that owns the hardware RX/TX buffers. Two line control implementations are provided: `SerialLineControl_ADS` (ADS-routed) and `SerialLineControl_Terminal` (direct hardware).

---

## SerialByteConnection

Sends and receives single bytes over a serial line.

Extends: `CyclicComponent`  
Implements: `I_SerialByteConnection`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, SerialLineController)` | Constructor | Binds to an `I_SerialLineControl` |
| `ByteData` | `BYTE` (Get) | Most recently received byte |
| `ByteDataRecieved` | `BOOL` (Get) | TRUE (once) when a new byte has arrived; auto-clears on read |
| `SendAByte(DataToSend)` | Method | Queues a byte for transmission |
| `Reset()` | Method | Clears RX buffer and resets counters |
| `CyclicLogic()` | Method | Must be called each scan |

### Example

```pascal
VAR
    LineCtrl : SerialLineControl_ADS('COM1 ADS');
    Conn     : SerialByteConnection('ByteConn', LineCtrl);
    RxByte   : BYTE;
END_VAR

LineCtrl.CyclicLogic();
Conn.CyclicLogic();

IF Conn.ByteDataRecieved THEN
    RxByte := Conn.ByteData;
END_IF

Conn.SendAByte(16#06);  // Send ACK
```

---

## SerialStringConnection

Sends and receives null-terminated strings over a serial line. Supports configurable prefix/suffix delimiters for framing.

Extends: `CyclicComponent`  
Implements: `I_SerialStringConnection`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name, SerialLineController)` | Constructor | Binds to an `I_SerialLineControl` |
| `StringData` | `T_MaxString` (Get) | Most recently received string |
| `StringRecieved` | `BOOL` (Get) | TRUE (once) when a complete string has arrived; auto-clears on read |
| `SendAString(DataToSend)` | Method | Queues a string for transmission |
| `SetSerialConfiguration(Prefix, Suffix)` | Method | Sets the start/end delimiters for string framing |
| `Reset()` | Method | Clears RX buffer and resets state |
| `CyclicLogic()` | Method | Must be called each scan |

### Example

```pascal
VAR
    LineCtrl : SerialLineControl_ADS('COM2 ADS');
    Conn     : SerialStringConnection('StringConn', LineCtrl);
    Response : T_MaxString;
END_VAR

// Frame strings with STX/ETX
Conn.SetSerialConfiguration(Prefix := '$02', Suffix := '$03');

LineCtrl.CyclicLogic();
Conn.CyclicLogic();

IF Conn.StringRecieved THEN
    Response := Conn.StringData;
    // Process response...
END_IF

Conn.SendAString('READ?');
```

---

## SerialLineControl_ADS

Routes serial communication through ADS to a TwinCAT serial port server. Use when the COM port is not on the local runtime.

Implements: `I_SerialLineControl`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name)` | Constructor | Standard name |
| `rxBuffer` | RX buffer reference | Consumed by `SerialByteConnection` / `SerialStringConnection` |
| `txBuffer` | TX buffer reference | Written to by send methods |
| `CyclicLogic()` | Method | Must be called each scan |

---

## SerialLineControl_Terminal

Direct serial communication using Beckhoff serial terminal hardware (e.g., EL6001/EL6002). Use when the COM port is accessed via EtherCAT terminal I/O mapping.

Implements: `I_SerialLineControl`

Same interface as `SerialLineControl_ADS`.
