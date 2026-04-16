# AdsReadWrite

Cyclic component that wraps TwinCAT ADS read/write function blocks. Provides both index-group/offset-based access and symbol-name-based access to variables on any ADS-reachable runtime.

Extends: `CyclicComponent`  
Implements: `I_AdsRW`

## Interface

| Member | Type | Description |
|--------|------|-------------|
| `FB_Init(Name)` | Constructor | Standard component name |
| `AmsNetId` | `T_AmsNetId` (Get/Set) | Target runtime AMS Net ID |
| `AmsPort` | `T_AmsPort` (Get/Set) | Target AMS port |
| `AdsComMode` | `E_AdsComMode` (Set) | Communication mode; default `eAdsComModeSecureCom` |
| `Busy` | `BOOL` (Get) | TRUE while any read or write is in progress |
| `Error` | `BOOL` (Get) | TRUE if any operation has faulted |
| `ErrorId` | `UDINT` (Get) | ADS error code of the most recent fault |
| `Read(IdxGrp, IdxOffset, pDestAddr, Length)` | Method | Triggers an index-group read |
| `ReadBySymbol(SymbolName, pDestAddr, Length)` | Method | Reads a variable by its symbol name |
| `Write(IdxGrp, IdxOffset, pSourceAddr, Length)` | Method | Triggers an index-group write |
| `WriteBySymbol(SymbolName, pSourceAddr, Length)` | Method | Writes a variable by its symbol name |
| `CyclicLogic()` | Method | Must be called each scan; services all pending operations |

## Notes

- Only one read and one write can be in-flight at a time; calls while `Busy` are silently dropped.
- `CyclicLogic()` resets the execute flags each scan — no manual reset required.

## Example

```pascal
VAR
    Ads      : AdsReadWrite('RemoteAds');
    MyValue  : DINT;
END_VAR

Ads.AmsNetId := '192.168.1.10.1.1';
Ads.AmsPort  := 851;

// Read a variable by symbol name
Ads.ReadBySymbol('MAIN.Counter', ADR(MyValue), SIZEOF(MyValue));
Ads.CyclicLogic();

// Write by symbol name
MyValue := 42;
Ads.WriteBySymbol('MAIN.Counter', ADR(MyValue), SIZEOF(MyValue));
Ads.CyclicLogic();
```
