# VFFS — Vertical Form Fill Seal Demo

Sample application that models a packaging machine end-to-end on top of the [`ApplicationBase`](../../ApplicationBase) framework. It composes **one Machine Module**, **three Equipment Modules**, and a custom `SealBar` component into a PackML-compliant application with recipe management, events, visitors, and an HMI.

## Module Hierarchy

```
VFFS  (MachineModule / PackMLModule)
├── Unwind       (EquipmentModule)  — film unwind axis
├── Sealer       (EquipmentModule)  — jaws + heated seal bar
└── PullWheels   (EquipmentModule)  — geared pull axes + cylinder
```

## Running

1. Open [`ApplicationDevelopment.sln`](../ApplicationDevelopment.sln) in TwinCAT XAE.
2. Activate the `VFFS` PLC task, log in, and start the runtime.
3. `MAIN.TcPOU` initializes the machine and runs `CyclicLogic()` every 100 ms; a `GetSystemTreeJsonHmiVisitor` walks the tree to build the HMI symbol map.

## Full Documentation

See [Documentation/VFFS.md](../../Documentation/VFFS.md) for the full breakdown — module/component responsibilities, production sequence, recipe structure, PackML states, and interfaces.
