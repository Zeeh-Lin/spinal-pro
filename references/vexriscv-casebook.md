# VexRiscv Casebook

Use this reference to turn an abstract SpinalHDL design question into concrete files to inspect in VexRiscv.

## Core Map

- Pipeline skeleton and default stageables:
  `VexRiscv/src/main/scala/vexriscv/VexRiscv.scala`
- Plugin host behavior, stage interconnect, and inserted data flow:
  `VexRiscv/src/main/scala/vexriscv/Pipeline.scala`
- Service contracts:
  `VexRiscv/src/main/scala/vexriscv/Services.scala`
- Plugin base type:
  `VexRiscv/src/main/scala/vexriscv/plugin/Plugin.scala`

Use these first when the question is "how does a pluginized SpinalHDL CPU hang together?"

## Configuration Examples

- Small, direct, productive baseline:
  `VexRiscv/src/main/scala/vexriscv/demo/GenSmallAndProductive.scala`
- Larger cached and MMU-enabled configuration:
  `VexRiscv/src/main/scala/vexriscv/demo/GenFull.scala`
- SoC integration, reset composition, APB peripherals, and bus plumbing:
  `VexRiscv/src/main/scala/vexriscv/demo/Murax.scala`

Use these when the question is "which features belong together?" or "what does a coherent top-level config look like?"

## Pattern Questions To File Anchors

- Custom instruction plugin:
  start with the "Add a custom instruction" section in `VexRiscv/README.md`
  then inspect `VexRiscv/src/main/scala/vexriscv/demo/GenCustomSimdAdd.scala`
- Custom CSR or software-visible extension:
  inspect `VexRiscv/src/main/scala/vexriscv/demo/CustomCsrDemoPlugin.scala`
- Frontend and branch prediction behavior:
  inspect `VexRiscv/src/main/scala/vexriscv/plugin/Fetcher.scala`
- Decode rules and decoded metadata:
  inspect `VexRiscv/src/main/scala/vexriscv/plugin/DecoderSimplePlugin.scala`
- Hazard and bypass policy:
  inspect `VexRiscv/src/main/scala/vexriscv/plugin/HazardSimplePlugin.scala`
- Branch staging and penalties:
  inspect `VexRiscv/src/main/scala/vexriscv/plugin/BranchPlugin.scala`
- Simple memory bus shapes and load/store timing:
  inspect `VexRiscv/README.md` sections for `IBusSimplePlugin` and `DBusSimplePlugin`
- Debug reset and debug bus behavior:
  inspect `VexRiscv/README.md` and `VexRiscv/src/main/scala/vexriscv/plugin/DebugPlugin.scala`

## Peripheral Tutorial Cases

For a smaller, easier-to-translate non-CPU example, inspect the GCD peripheral tutorial:

- Narrative and design intent:
  `VexRiscv/doc/gcdPeripheral/README.md`
- Shared control/data bundle and top-level wiring:
  `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDTop.scala`
- Datapath:
  `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDData.scala`
- FSM control path:
  `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDCtrl.scala`
- APB register bank integration:
  `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/Apb3GCDCtrl.scala`
- Focused simulation:
  `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDTopSim.scala`

Use this cluster when the task is about:

- datapath versus control separation
- `Bundle with IMasterSlave`
- APB register mapping
- small standalone simulation with a software oracle

## Do Not Cargo-Cult

Before copying a VexRiscv pattern, verify:

- the destination design has the same timing and stage assumptions
- the same reset and domain assumptions apply
- the same service dependencies exist
- the copied bus contract matches the destination environment
- the copied config knobs are still meaningful outside VexRiscv

Treat VexRiscv as a mature pattern library, not as a substitute for official SpinalHDL semantics.
