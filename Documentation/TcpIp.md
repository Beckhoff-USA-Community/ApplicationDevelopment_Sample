# TCP/IP Components

## TcpIpConnection

Cyclic component that manages a persistent TCP/IP client connection to a remote server. Handles automatic reconnection, queued command sending (normal and priority), and response routing to registered subscribers.

Extends: `CyclicComponent`  
Implements: `I_TcpIpConnection`, `I_Initializable`

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name)` | Constructor | Standard component name |
| `IP` | `T_IPv4Addr` (Set) | Remote server IP address; default `'127.0.0.1'` |
| `Port` | `UDINT` (Set) | Remote server port |
| `ConnectionState` | `E_SocketConnectionState` (Get) | Current socket state (`eSOCKET_CONNECTED`, etc.) |
| `Initialized` | `BOOL` (Get/Set) | TRUE after `Initialize()` completes |
| `ReconnectionTime` | `TIME` (Set) | Delay before reconnect attempt after disconnect; default `T#10S` |
| `PollTime` | `TIME` (Set) | How often to poll for incoming data; default `T#100MS` |
| `ReceiveTimeout` | `TIME` (Set) | Time before a stale connection is considered lost; default `T#50S` |
| `SendInterval` | `TIME` (Set) | Minimum time between sends; default `T#100MS` |
| `Socket` | `T_HSOCKET` (Get) | Raw socket handle |
| `Initialize()` | Method | Closes all sockets and prepares for a fresh connection |
| `SendCommand(Command)` | Method | Queues a payload in the normal send buffer (up to 100 entries) |
| `SendPriorityCommand(Command)` | Method | Queues a payload in the priority send buffer (sent before normal queue) |
| `RegisterCommand(Command)` | Method | Registers an `I_TcpIpSubscription` to receive response callbacks |
| `CyclicLogic()` | Method | Must be called each scan; manages connection, send, and receive state machine |

### Notes

- Connection is maintained automatically; if the socket drops, the component waits `ReconnectionTime` then reconnects.
- Responses are broadcast to all registered `I_TcpIpSubscription` instances via `UpdateCommandResult()`.
- Both send buffers hold up to 100 `ST_TcpIpPayload` entries each. Priority buffer is drained first.

### Example

```pascal
VAR
    Conn    : TcpIpConnection('BarcodeScannerConn');
    Payload : ST_TcpIpPayload;
END_VAR

Conn.IP   := '192.168.1.50';
Conn.Port := 2112;

Conn.CyclicLogic();

IF Conn.ConnectionState = eSOCKET_CONNECTED THEN
    // Build and queue a command
    Payload.Size := 5;
    MEMCPY(ADR(Payload.Data), ADR('READ$00'), Payload.Size);
    Conn.SendCommand(ADR(Payload));
END_IF
```

---

## TcpIpCommandResultFilter

Implements `I_TcpIpSubscription`. Registers with a `TcpIpConnection` and filters incoming response payloads, forwarding only those that match a configured pattern or command identifier to the application layer.

### Interface

| Member | Type | Description |
|--------|------|-------------|
| `UpdateCommandResult(pPayload)` | Method | Called by `TcpIpConnection` for each received payload |
| `Result` | access to matched response | Application reads the matched/filtered result here |

Register with the connection using `Conn.RegisterCommand(MyFilter)`.
