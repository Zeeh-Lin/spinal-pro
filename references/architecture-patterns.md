# Architecture Patterns

Use this reference for medium-to-large SpinalHDL design work: decomposition, plugin systems, SoC integration, and parameterized structure.

## Choose The Lightest Viable Pattern

- Use a single `Component` for small local logic with little reuse pressure.
- Use control-path plus data-path decomposition for algorithmic blocks with clear sequencing.
- Use configuration objects plus multiple subcomponents for reusable peripherals or subsystems.
- Use plugins and services only when extensibility, feature swapping, or cross-cutting injection is a real requirement.

## Plugin-Oriented Composition

SpinalHDL supports plugin-based composition, and VexRiscv is the most mature local example.

Carry these rules across:

- keep the host small and let plugins inject behavior
- separate negotiation/setup concerns from hardware build concerns
- expose narrow services instead of reaching into unrelated internals
- make staging explicit when data appears in one stage and is consumed in another

Primary anchors:

- `SpinalDoc-RTD/source/SpinalHDL/Libraries/Misc/service_plugin.rst`
- `VexRiscv/src/main/scala/vexriscv/plugin/Plugin.scala`
- `VexRiscv/src/main/scala/vexriscv/Pipeline.scala`
- `VexRiscv/src/main/scala/vexriscv/Services.scala`
- `VexRiscv/src/main/scala/vexriscv/VexRiscv.scala`

## Stage-Coupled CPU Patterns

For CPU-like work, do not copy a plugin in isolation.
Trace at least:

- where the result is produced
- where the result is advertised as bypassable
- which service the plugin depends on
- which frontend timing assumptions it expects
- which reset and exception contracts it relies on

Primary anchors:

- `VexRiscv/README.md`
- `VexRiscv/src/main/scala/vexriscv/plugin/DecoderSimplePlugin.scala`
- `VexRiscv/src/main/scala/vexriscv/plugin/Fetcher.scala`
- `VexRiscv/src/main/scala/vexriscv/plugin/HazardSimplePlugin.scala`
- `VexRiscv/src/main/scala/vexriscv/plugin/BranchPlugin.scala`

## Configuration As Architecture

Treat the configuration object as part of the architecture, not just convenience glue.

- Group related knobs into a config type.
- Keep example configurations that show intended tradeoffs.
- Review interactions between cache, MMU, regfile timing, branch timing, and hazard policy together.

Primary anchors:

- `VexRiscv/src/main/scala/vexriscv/demo/GenSmallAndProductive.scala`
- `VexRiscv/src/main/scala/vexriscv/demo/GenFull.scala`
- `VexRiscv/src/main/scala/vexriscv/demo/Murax.scala`

## SoC Integration Pattern

For a peripheral or subsystem integrated into a larger SoC:

- keep bus-facing logic separate from the computational core
- put memory map policy in one place
- isolate reset and clock-domain adaptation
- connect software-visible behavior to explicit status and control registers

Primary anchors:

- `VexRiscv/doc/gcdPeripheral/README.md`
- `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/Apb3GCDCtrl.scala`
- `VexRiscv/src/main/scala/vexriscv/demo/Murax.scala`

## Pattern Translation Guidance

Translate patterns, not code.

- Borrow decomposition style from VexRiscv.
- Borrow handshake and bus modeling style from official SpinalHDL libraries.
- Borrow stage and service reasoning only if the target design really has comparable lifecycle and timing structure.
- Rewrite names, interfaces, and assumptions to match the destination repository.

## Anti-Patterns

- Copying a VexRiscv plugin without its dependent service contracts
- Copying branch, hazard, or debug logic without matching reset and stage timing
- Hiding structural changes behind ambiguous booleans
- Letting optional features create undriven or partially assigned signals
- Using plugin abstraction where a plain subcomponent would be clearer
