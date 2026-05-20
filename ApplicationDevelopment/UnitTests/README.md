# UnitTests

TwinCAT PLC project that exercises the function blocks in [`ApplicationBase`](../../ApplicationBase). Each test POU lives under `UnitTests/` and is invoked from `MAIN.TcPOU`. The project doubles as executable documentation showing how each building block is intended to be used.

## Running

Tests run inside the TwinCAT runtime — there is no CLI test runner.

- **Run all tests** — open [`ApplicationDevelopment.sln`](../ApplicationDevelopment.sln), activate the `UnitTests` PLC task, log in, and start the runtime. `MAIN.TcPOU` calls every test in turn.
- **Run a single test** — comment out all other test function calls in `MAIN.TcPOU`, leaving only the one you want (e.g., `Component_TEST()`).
- **Inspect results** — open the global `GlobalTestSuite.TestSuite` in the TwinCAT online view. Each test reports pass/fail with assertion messages.

## Authoring a Test

```pascal
// Every test guards execution against the TestSuite:
IF NOT TestSuite.Test(__POUNAME()).ExecuteTest() THEN RETURN; END_IF

// Assertions:
TestSuite.AssertEqual(actual, expected, 'message');
```

Mockups for components and modules live in [`_Mockups/`](_Mockups).

## Task Configuration

- Task name: `UnitTests`
- Priority: 20
- Cycle time: 100 ms
- AMS Port: 350
