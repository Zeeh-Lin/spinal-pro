# Architecture Patterns

Medium-to-large SpinalHDL design work: decomposition, pipelining, plugin
systems, SoC integration, and parameterized structure.

Target version: SpinalHDL 1.10+ (the `spinal.lib.misc.pipeline` and
`spinal.lib.misc.plugin` libraries are assumed available).

## Choose The Lightest Viable Pattern

| Situation | Pattern |
| --- | --- |
| Small local logic, no reuse pressure | Single `Component` |
| Algorithmic block with clear sequencing | Control-path + data-path with a `Bundle` interconnect |
| Reusable peripheral or subsystem | Config `case class` + multiple subcomponents |
| Multi-stage data pipeline with possibly variable depth | `spinal.lib.misc.pipeline.{Node, StageLink, ...}` |
| Cross-cutting feature injection, optional extensions | `FiberPlugin` + `PluginHost` |

Escalate one tier only when the lower tier creates real friction. Plugin
abstractions are powerful but reviewers pay for them, so do not reach for
them before structural decomposition has been tried.

## Control-Path + Data-Path Decomposition

Use this when the block has a clearly separable algorithm: an FSM walks
through phases while a datapath computes per phase.

```scala
case class GcdBundle() extends Bundle with IMasterSlave {
  val start = Bool()
  val done  = Bool()
  val a, b  = UInt(16 bits)
  val res   = UInt(16 bits)

  override def asMaster(): Unit = {
    out(start, a, b)
    in(done, res)
  }
}

class GcdData extends Component {
  val io = new Bundle {
    val ctl = slave (GcdBundle())
  }
  // datapath: subtract / compare / register updates only
}

class GcdCtrl extends Component {
  val io = new Bundle {
    val ctl = master (GcdBundle())
  }
  // FSM only; never compute arithmetic here
}

class GcdTop extends Component {
  val data = new GcdData
  val ctrl = new GcdCtrl
  ctrl.io.ctl <> data.io.ctl
}
```

Trade-off: more files, but two reviewers can change the data path and the
control path independently without merge friction. Reference real-world
shape: `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/*` ·
https://github.com/SpinalHDL/VexRiscv/tree/master/doc/gcdPeripheral

## Pipeline Library (`spinal.lib.misc.pipeline`)

The recommended way to build a pipelined data path is the official
pipeline library. It generates the `valid`/`ready` arbitration and the
register stages from a small description.

```scala
import spinal.lib.misc.pipeline._

class StreamPlus3 extends Component {
  val io = new Bundle {
    val up   = slave  Stream(UInt(16 bits))
    val down = master Stream(UInt(16 bits))
  }

  val n0, n1, n2 = Node()
  val s01 = StageLink(n0, n1)
  val s12 = StageLink(n1, n2)

  val IN  = Payload(UInt(16 bits))
  val OUT = Payload(UInt(16 bits))

  n0.driveFrom(io.up)((node, p) => node(IN) := p)
  n1(OUT) := n1(IN) + 0x1200
  n2.driveTo(io.down)((p, node) => p := node(OUT))

  Builder(s01, s12)
}
```

Key concepts:
- `Node` — represents one stage; holds the local `valid`/`ready`/`cancel`.
- `Payload` — a typed key referring to "this signal on this stage". The
  builder pipelines occurrences automatically.
- `StageLink` — registered link with arbitration. Also `DirectLink`,
  `S2mLink`, `CtrlLink` for control-flow stages with halt / throw / bypass.
- `Builder(...)` or `pip.build()` generates the arbitration hardware.

For CPU-like stages with hazard/flush logic use `CtrlLink` (provides
`haltWhen`, `throwWhen`, `forgetOneWhen`, `bypass(Payload)`) and a
`StageCtrlPipeline` to compose them.

When to use this instead of a hand-rolled pipeline:
- the stage count is parameterized or might change
- the same `Payload` lives on many stages and should be pipelined automatically
- you want a uniform arbitration model across all stages

When to *not* use it:
- the block is two registers and an output: hand wiring is clearer
- the design must match an exact legacy stage topology you cannot redraw

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Libraries/Pipeline/introduction.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/Pipeline/introduction.html

## Plugin Composition (`spinal.lib.misc.plugin`)

Reach for plugins when the component must be extended *from outside* — by
swapping multipliers, adding optional extensions, or assembling a
distributed system where each feature owns its services.

```scala
import spinal.core._
import spinal.core.fiber._
import spinal.lib.misc.plugin._

class StatePlugin extends FiberPlugin {
  val logic = during build new Area {
    val signal = Reg(UInt(32 bits)) init(0)
  }
}

class DriverPlugin extends FiberPlugin {
  var incrementBy = 0
  val retainer = Retainer()

  val logic = during build new Area {
    val sp = host[StatePlugin].logic.get
    retainer.await()
    sp.signal := sp.signal + incrementBy
  }
}

class Cpu extends Component {
  val host = new PluginHost()
}

object Top extends App {
  SpinalVerilog {
    val cpu = new Cpu
    cpu.host.asHostOf(new StatePlugin, new DriverPlugin)
    cpu
  }
}
```

Elaboration model:
- `during setup` runs first; plugins use it to look up neighbors and
  install retainers / locks.
- `during build` runs after every setup phase has returned; this is the
  only place hardware is materialized.
- `host[X]` blocks until plugin `X` has finished its current phase.

Recommendations:
- keep the host component small — almost no hardware in the top file
- expose narrow services (`host[X].logic.someSignal`) rather than reaching
  into other plugins' internals
- when a plugin must wait for input from peers, gate its build body with
  `retainer.await()` so callers can prep state before hardware appears
- avoid leaving locked retainers if a peer is missing — that hangs
  elaboration

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Libraries/Misc/service_plugin.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/Misc/service_plugin.html

## Configuration As Architecture

Treat the configuration object as part of the architecture, not just glue.

```scala
case class CpuCfg(
    iCache: Boolean = true,
    dCache: Boolean = true,
    mmu: Boolean    = false,
    mulDivLatency: Int = 3
) {
  require(mulDivLatency >= 1)
  require(!mmu || (iCache && dCache),
    "MMU requires both instruction and data caches")
}
```

Rules:
- group related knobs into a single config type
- enforce coherent combinations with `require`, not comments
- check the config invariants once at the top of the component, not
  scattered through children
- ship a small set of canonical config instances (`small`, `default`,
  `withMmu`) and reference them in tests

Mature examples: `VexRiscv/src/main/scala/vexriscv/demo/GenSmallAndProductive.scala`,
`GenFull.scala`, and `Murax.scala` ·
https://github.com/SpinalHDL/VexRiscv/tree/master/src/main/scala/vexriscv/demo

## SoC Integration Pattern

For a peripheral integrated into a larger SoC:

- keep the bus-facing logic separate from the computational core
- centralize the memory map in one file (a `case class MemoryMap` or
  similar)
- isolate reset and clock-domain adaptation in a dedicated `ClockingArea`
- connect software-visible behavior to explicit status / control / event
  registers via `BusSlaveFactory`
- avoid leaking internal counters or FSM states into the bus interface
  without a register stage

Reference shape: `VexRiscv/src/main/scala/vexriscv/demo/Murax.scala` ·
https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/demo/Murax.scala
together with the GCD peripheral example.

## Pattern Translation Guidance

Translate ideas, not lines. When you reach for a pattern from VexRiscv,
NaxRiscv, or any external project, verify these match:

| Question | If "no" |
| --- | --- |
| Same stage count and timing assumptions? | Re-stage or pick a smaller pattern |
| Same reset semantics and domain composition? | Adapt the reset network first |
| Same service dependencies (does the source plugin require peers you don't have)? | Provide stubs or remove the dependency |
| Same bus contract (latency, response timing)? | Wrap or rewrite the bus glue |
| Same config knobs still meaningful? | Drop or rename the knobs to suit the new design |

Always rewrite names, interfaces, and assumptions to match the destination
repo. Borrowed code that keeps source-tree names is the easiest signal of
unsafe copy-paste.

## Anti-Patterns

- Copying a VexRiscv plugin without its service contracts and stage timing
  assumptions.
- Copying branch / hazard / debug logic without matching reset and stage
  composition.
- Hiding structural changes behind ambiguous booleans (`mode: Int = 0`)
  rather than typed enums or sealed traits.
- Letting an optional feature leave some signals undriven — this becomes
  a `LATCH DETECTED` or `NO DRIVER ON` at generation time.
- Reaching for plugins where a plain subcomponent would have been clearer.
- Building a hand-rolled pipeline when `spinal.lib.misc.pipeline` would
  have generated the same arbitration in a fraction of the code.
- Building a custom service framework when `PluginHost` + `Retainer` would
  cover it.
