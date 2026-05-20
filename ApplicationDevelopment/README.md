# ApplicationDevelopment

TwinCAT solution that hosts three PLC projects built on the [`ApplicationBase`](../ApplicationBase) framework.

Open [`ApplicationDevelopment.sln`](ApplicationDevelopment.sln) in **TwinCAT 3 XAE** to load all three projects.

## Projects

| Project | What it is |
|---------|-----------|
| [Template](Template) | Minimal, pre-wired starting point for a new application. Use it as the scaffold when starting a new machine. See [Documentation/Template.md](../Documentation/Template.md). |
| [VFFS](VFFS) | Complete demo modelling a Vertical Form Fill Seal packaging machine — one Machine Module, three Equipment Modules, and components composed end-to-end. See [Documentation/VFFS.md](../Documentation/VFFS.md). |
| [UnitTests](UnitTests) | PLC project that exercises every major building block in `ApplicationBase`. Doubles as executable documentation for how each function block is intended to be used. |

## Building & Running

1. Open `ApplicationDevelopment.sln` in TwinCAT XAE.
2. Choose a project's PLC task as the active task (only one PLC task runs per runtime).
3. Build → Activate Configuration → Login → Run on the target controller or simulator.

## Running the Unit Tests

The unit tests run inside the TwinCAT runtime — there is no CLI test runner.

- Run all tests: download `UnitTests` and let it run; `UnitTests/MAIN.TcPOU` orchestrates every test call.
- Run a single test: comment out the other test function calls in `MAIN.TcPOU`.
- Inspect results: open the global `GlobalTestSuite.TestSuite` in the TwinCAT online view.
