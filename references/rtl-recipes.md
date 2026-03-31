# RTL Recipes

Use this reference for implementation patterns that recur in SpinalHDL code.

## Contents

- Basic Component Recipe
- Bundle And Interface Recipe
- Algorithmic Block Recipe
- Stream And Flow Recipe
- Register Bank And Bus Mapping Recipe
- Parameterization Recipe
- Clock-Domain Recipe
- Validation Hook Recipe

## Basic Component Recipe

- Start from a small `Component` with a clear `io` bundle.
- Give combinational outputs a safe default before conditional overrides.
- Introduce registers only where state is required.
- Keep semantic-heavy helper logic near the signal it affects.

Primary anchors:

- `SpinalDoc-RTD/source/SpinalHDL/Semantic/assignments.rst`
- `SpinalDoc-RTD/source/SpinalHDL/Sequential logic/registers.rst`

## Bundle And Interface Recipe

- Use `case class ... extends Bundle` for typed interfaces.
- Use `IMasterSlave` when the interface has a clear master/slave viewpoint.
- Implement `asMaster()` once, then connect with `<>`.
- Use `asBits` and `assignFromBits` only when packing or unpacking is genuinely needed.

Primary anchors:

- `SpinalDoc-RTD/source/SpinalHDL/Data types/bundle.rst`
- `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDTop.scala`

## Algorithmic Block Recipe

For software-like algorithms with loops or branch-heavy control:

- split control path and data path
- define an explicit interconnect bundle between them
- keep datapath arithmetic and compares in one block
- keep sequencing in an FSM block
- expose only top-level control signals such as `valid`, `ready`, or `start`, `done`

Primary anchors:

- `SpinalDoc-RTD/source/SpinalHDL/Libraries/fsm.rst`
- `VexRiscv/doc/gcdPeripheral/README.md`
- `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDCtrl.scala`
- `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDData.scala`

## Stream And Flow Recipe

- Use `Stream` for backpressured command, request, response, or FIFO traffic.
- Use `Flow` for event-like outputs where the consumer cannot stall.
- Prefer standard helpers such as `.queue`, `.m2sPipe`, `.s2mPipe`, and `.throwWhen` over custom glue when they express the intent.
- When defining a custom bus, decide explicitly whether the response side is `Stream`, `Flow`, or a fixed-latency plain bundle.

Primary anchors:

- `SpinalDoc-RTD/source/SpinalHDL/Libraries/stream.rst`
- `SpinalDoc-RTD/source/SpinalHDL/Libraries/flow.rst`
- `VexRiscv/README.md` sections for `IBusSimplePlugin`, `DBusSimplePlugin`, and cached buses

## Register Bank And Bus Mapping Recipe

- Prefer `BusSlaveFactory` or the bus-specific factory instead of hand-decoding addresses.
- Separate command strobes, sticky status, and readback state.
- For read-clear semantics, make the clear event explicit.
- For asynchronous status crossing into the bus clock domain, synchronize deliberately instead of reading live combinational values.

Primary anchors:

- `SpinalDoc-RTD/source/SpinalHDL/Libraries/bus_slave_factory.rst`
- `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/Apb3GCDCtrl.scala`
- `VexRiscv/src/main/scala/vexriscv/demo/Murax.scala`

## Parameterization Recipe

- Use config case classes for multi-field structural choices.
- Use `generate` or `ifGen` to create or omit optional hardware cleanly.
- Keep parameter effects local and obvious.
- Prefer typed parameters over boolean soup when a subsystem has multiple related knobs.

Primary anchors:

- `SpinalDoc-RTD/source/SpinalHDL/Data types/bundle.rst`
- `VexRiscv/src/main/scala/vexriscv/demo/Murax.scala`
- `VexRiscv/src/main/scala/vexriscv/demo/GenSmallAndProductive.scala`
- `VexRiscv/src/main/scala/vexriscv/demo/GenFull.scala`

## Clock-Domain Recipe

- Create a dedicated `ClockDomain` only when the design really crosses or splits domains.
- Instantiate state inside the intended `ClockingArea`.
- Keep domain conversion and reset adaptation explicit and reviewable.

Primary anchors:

- `SpinalDoc-RTD/source/SpinalHDL/Structuring/clock_domain.rst`
- `SpinalDoc-RTD/source/SpinalHDL/Examples/Simple ones/pll_resetctrl.rst`
- `VexRiscv/src/main/scala/vexriscv/demo/Murax.scala`

## Validation Hook Recipe

- Add a small Scala simulation harness when the block is algorithmic or interface-heavy.
- Use a software model or scoreboard when a reference function is easy to compute.
- Exercise reset, idle, repeated transactions, and corner values.

Primary anchors:

- `SpinalDoc-RTD/source/SpinalHDL/Simulation/index.rst`
- `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDTopSim.scala`
