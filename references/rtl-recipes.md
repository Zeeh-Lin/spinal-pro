# RTL Recipes

Self-contained implementation patterns. Every recipe shows minimal Scala
along with the doc anchor (local path + URL) that the agent should reach
for if more depth is needed.

Target version: SpinalHDL 1.10+.

## Contents

- Component skeleton
- `Bundle` and `IMasterSlave` interface
- Register patterns and reset
- `Stream` handshake and toolkit
- `Flow` events
- `StateMachine` (`spinal.lib.fsm`)
- `Mem` (RAM/ROM) patterns
- `BusSlaveFactory` register bank
- Clock domains and CDC helpers
- Parameterization
- `SpinalConfig` and generation entry points

## Component Skeleton

```scala
import spinal.core._

class Adder(width: Int) extends Component {
  val io = new Bundle {
    val a, b = in  UInt(width bits)
    val sum  = out UInt(width + 1 bits)
  }

  io.sum := io.a.resize(width + 1) + io.b.resize(width + 1)
}

object AdderGen extends App {
  SpinalVerilog(new Adder(8))
}
```

Conventions worth keeping:
- group ports under a single `io: Bundle`
- give every combinational output a safe default before any conditional override
- declare registers in the area where they should belong to their target clock domain

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Structuring/components_hierarchy.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Structuring/components_hierarchy.html

## Bundle And IMasterSlave Interface

```scala
case class Handshake(width: Int) extends Bundle with IMasterSlave {
  val valid   = Bool()
  val ready   = Bool()
  val payload = Bits(width bits)

  override def asMaster(): Unit = {
    out(valid, payload)
    in(ready)
  }
}

class Bridge(width: Int) extends Component {
  val io = new Bundle {
    val up   = slave  (Handshake(width))
    val down = master (Handshake(width))
  }
  io.down <> io.up
}
```

Rules of thumb:
- `<>` only works correctly when `asMaster` (or built-in directions) are set
- `master(MyBundle())` / `slave(MyBundle())` resolve direction from `asMaster`
- prefer `case class` over `class` so SpinalHDL can `cloneOf` it without help

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Data types/bundle.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Data%20types/bundle.html

## Register Patterns And Reset

```scala
val r1 = Reg(UInt(8 bits)) init(0)
val r2 = RegInit(U"8'h00")
val r3 = RegNext(io.in)            // samples io.in every cycle
val r4 = RegNextWhen(io.in, io.en) // samples io.in only when io.en

// Per-field reset on a register bundle
case class StatusFlags() extends Bundle {
  val ready, error = Bool()
  val code         = Bits(4 bits)
}

val st = Reg(StatusFlags())
st.ready init(False)
st.error init(False)
// st.code intentionally has no init: synthesizer can drop reset on it
```

Common gotcha: a register declared inside the default clock domain, but
intended to live in another one, will silently misbehave. Always create
registers inside the right `ClockingArea`.

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Sequential logic/registers.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Sequential%20logic/registers.html

## Stream Handshake And Toolkit

```scala
import spinal.lib._

class StreamPlumbing extends Component {
  val io = new Bundle {
    val a   = slave  Stream(UInt(8 bits))
    val b   = master Stream(UInt(8 bits))
  }

  // 4-deep FIFO. `.queue(n)` is latency 2; the additional `<-<` adds one
  // m2sPipe stage (latency 1) for a total of 3 cycles forward latency.
  // Drop the `<-<` to keep it at 2.
  io.b <-< io.a.queue(size = 4)
}
```

Cheat sheet of helpers (`spinal.lib._`):

| Need | Idiom |
| --- | --- |
| FIFO buffer | `src.queue(size)` or `StreamFifo(dataType, depth)` |
| Forward register stage | `src.m2sPipe()` or `src.stage()` or `dst <-< src` |
| Backward register stage | `src.s2mPipe()` or `dst </< src` |
| Full handshake register | `dst <-/< src` |
| Drop matching transactions | `src.throwWhen(cond)` |
| Hold transactions back | `src.haltWhen(cond)` |
| Replace payload | `src.translateWith(newPayload)` |
| Function-style transform | `src.map(t => newT)` |
| Cross clock domain | `StreamFifoCC(t, depth, pushClk, popClk)` or `StreamCCByToggle(...)` |
| N→1 arbitration | `StreamArbiterFactory.roundRobin.build(...)` |
| 1→N dispatch | `StreamDemux(input, select, portCount)` |

Bigger example:

```scala
class StreamCdcWithFilter[T <: Data](dataType: HardType[T], depth: Int,
                                     srcCd: ClockDomain, dstCd: ClockDomain) extends Component {
  val io = new Bundle {
    val src = slave  Stream(dataType)
    val dst = master Stream(dataType)
  }

  val fifo = StreamFifoCC(
    dataType  = dataType,
    depth     = depth,
    pushClock = srcCd,
    popClock  = dstCd
  )
  fifo.io.push << io.src
  fifo.io.pop  >> io.dst
}
```

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Libraries/stream.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html

## Flow Events

```scala
class EdgeReporter extends Component {
  val io = new Bundle {
    val sample = in  Bool()
    val rising = master Flow(NoData())
  }
  val prev = RegNext(io.sample) init(False)
  io.rising.valid := io.sample && !prev
}
```

Only reach for `Flow` when the consumer cannot stall (one-shot events, raw
write strobes). If a sink may exert backpressure, use `Stream`.

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Libraries/flow.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/flow.html

## StateMachine (`spinal.lib.fsm`)

```scala
import spinal.lib.fsm._

class Pulser(width: Int) extends Component {
  val io = new Bundle {
    val start = in  Bool()
    val done  = out Bool()
    val pulse = out Bool()
  }

  io.done  := False
  io.pulse := False

  val fsm = new StateMachine {
    val idle: State = new State with EntryPoint {
      whenIsActive {
        when(io.start) { goto(busy) }
      }
    }

    val busy: State = new State {
      val count = Reg(UInt(width bits)) init(0)
      onEntry { count := 0 }
      whenIsActive {
        io.pulse := True
        count := count + 1
        when(count === count.maxValue) { goto(finish) }
      }
    }

    val finish: State = new State {
      whenIsActive {
        io.done := True
        goto(idle)
      }
    }
  }
}
```

- States are `val`s with explicit `: State` type so the FSM can reference
  forward names.
- `EntryPoint` or `setEntry(state)` selects the reset state.
- `whenIsActive` is the per-cycle body; `onEntry` / `onExit` are pulses.
- `isActive(state)` and `isEntering(state)` expose status outside the FSM.

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Libraries/fsm.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/fsm.html

## Mem (RAM / ROM)

```scala
// Single-port read/write, synchronous read
val mem = Mem(Bits(32 bits), wordCount = 256)
mem.write(
  enable  = io.wrEna,
  address = io.wrAddr,
  data    = io.wrData
)
io.rdData := mem.readSync(
  enable  = io.rdEna,
  address = io.rdAddr
)

// Simple dual-port (1 write, 1 sync read) — same code as above, the read
// and the write live on separate addresses, the inferred macro depends on
// the target FPGA.

// Single read/write port. `readWriteSync` returns the read data only;
// the write happens when `enable && write`, the read happens when `enable`.
io.p0.rdata := mem.readWriteSync(
  address = io.p0.addr,
  data    = io.p0.wdata,
  enable  = io.p0.en,
  write   = io.p0.we
)
// For true dual-port, instantiate `readWriteSync` twice (once per port).

// ROM seeded from an initial array
val rom = Mem(UInt(8 bits), initialContent = (0 until 16).map(U(_, 8 bits)).toArray)
io.romData := rom.readAsync(io.romAddr)
```

Gotchas:
- Memory ports are explicit; do not use Verilog-style coding templates.
- An `enable` passed into `readSync` is **not** auto-ANDed with the
  surrounding `when` condition — combine them yourself.
- The default emitted Verilog uses `readFirst` semantics; for `writeFirst`
  you must enable automatic memory blackboxing (`SpinalConfig.addStandardMemBlackboxing`).

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Sequential logic/memory.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Sequential%20logic/memory.html

## BusSlaveFactory Register Bank

`BusSlaveFactory` is the canonical way to expose a register bank on APB3,
APB4, AXI-Lite, AXI4, Wishbone, AvalonMM, BMB, Tilelink, or
`PipelinedMemoryBus`. Same API, different bus.

```scala
import spinal.lib.bus.amba3.apb._

class GpioApb3(width: Int, apbConfig: Apb3Config) extends Component {
  val io = new Bundle {
    val apb  = slave (Apb3(apbConfig))
    val gpio = out  Bits(width bits)
    val ints = in   Bits(width bits)
  }

  val ctrl = new Apb3SlaveFactory(io.apb)

  val output = Reg(Bits(width bits)) init(0)
  io.gpio := output

  ctrl.readAndWrite(output, address = 0x00, bitOffset = 0)
  ctrl.read(io.ints,        address = 0x04, bitOffset = 0)

  // Sticky bit set when ints has a rising edge, cleared on read
  val prev    = RegNext(io.ints) init(0)
  val pending = Reg(Bits(width bits)) init(0)
  pending := pending | (io.ints & ~prev)
  ctrl.onRead(0x08) { pending := 0 }
  ctrl.read(pending, address = 0x08, bitOffset = 0)
}
```

Common building blocks: `readAndWrite`, `drive`, `driveFlow`, `read`,
`createReadAndWrite`, `doBitsAccumulationAndClearOnRead`,
`createAndDriveFlow`, `readStreamNonBlocking`.

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Libraries/bus_slave_factory.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/bus_slave_factory.html

## Clock Domains And CDC Helpers

```scala
class TwoDomainCounter extends Component {
  val io = new Bundle {
    val clkA, rstA = in  Bool()
    val clkB, rstB = in  Bool()
    val pulseA     = in  Bool()
    val countB     = out UInt(8 bits)
  }

  val cdA = ClockDomain(io.clkA, io.rstA,
    config = ClockDomainConfig(resetKind = ASYNC, resetActiveLevel = HIGH))
  val cdB = ClockDomain(io.clkB, io.rstB,
    config = ClockDomainConfig(resetKind = ASYNC, resetActiveLevel = HIGH))

  // Move a 1-bit pulse from cdA to cdB using a toggle + 2FF synchronizer.
  // Positional args here because the constructor parameter names are not
  // covered by the official RTD docs; check spinal.lib source if you need
  // the named-arg form for your SpinalHDL version.
  val pulseCC = PulseCCByToggle(io.pulseA, cdA, cdB)

  val countArea = new ClockingArea(cdB) {
    val count = Reg(UInt(8 bits)) init(0)
    when(pulseCC) { count := count + 1 }
  }
  io.countB := countArea.count
}
```

Cheat sheet (call with positional args unless your SpinalHDL version is
known to accept the named form):

| Need | Helper |
| --- | --- |
| Sync a 1-bit level into the local domain | `BufferCC(signal, init = ...)` |
| Move a 1-bit pulse between domains | `PulseCCByToggle(pulse, srcCd, dstCd)` |
| Cross a `Flow` cheaply (lossy on bursts) | `FlowCCUnsafeByToggle(flow, srcCd, dstCd)` |
| Cross a `Stream` safely (low bandwidth) | `StreamCCByToggle(stream, srcCd, dstCd)` |
| Cross a `Stream` with a CC FIFO | `StreamFifoCC(dataType, depth, srcCd, dstCd)` |
| Cross a clock-domain reset for a peripheral | `ResetCtrl.asyncAssertSyncDeassert(input, clockDomain)` |

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Structuring/clock_domain.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Structuring/clock_domain.html
CDC helpers reference: `SpinalDoc-RTD/source/SpinalHDL/Libraries/utils.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/utils.html

## Parameterization

```scala
case class UartConfig(
    dataWidth: Int      = 8,
    fifoDepth: Int      = 16,
    withParity: Boolean = false
) {
  require(dataWidth >= 5 && dataWidth <= 9)
}

class Uart(cfg: UartConfig) extends Component {
  val io = new Bundle {
    val tx = master Stream(Bits(cfg.dataWidth bits))
    val rx = master Stream(Bits(cfg.dataWidth bits))
    val parityErr = cfg.withParity generate (out Bool())
  }
  // body...
}
```

- Use `case class` configs, validate invariants with `require(...)`.
- `generate` (Bundle) and `if (...)` (Scala) are the two ways to omit
  optional hardware; pick the one that matches whether the gating happens
  at Bundle level or top-level structural level.
- Keep parameter effects local: if turning off a feature requires touching
  three files, the abstraction is wrong.

## SpinalConfig And Generation Entry Points

```scala
import spinal.core._

object AdderGen extends App {
  val report = SpinalConfig(
    targetDirectory               = "rtl",
    defaultClockDomainFrequency   = FixedFrequency(100 MHz),
    defaultConfigForClockDomains  = ClockDomainConfig(
      resetKind        = ASYNC,
      resetActiveLevel = HIGH
    ),
    onlyStdLogicVectorAtTopLevelIo = true
  ).generateVerilog(new Adder(8))

  println(s"Top is ${report.toplevelName}")
}
```

- `SpinalConfig.generateVerilog(...)` / `.generateVhdl(...)` are the
  recommended entry points; `SpinalVerilog(component)` is a shortcut.
- `targetDirectory` defaults to the current working directory — set it
  explicitly so generated files are easy to locate.
- `defaultClockDomainFrequency` only annotates the clock; it does not
  create a PLL.
- Generated files land in `${targetDirectory}/${toplevelName}.v` /`.sv`
  /`.vhd` plus a `.bin` report file alongside.

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Structuring/clock_domain.rst` (for
`ClockDomainConfig`) and the Getting Started chapter at
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Getting%20Started/index.html

## Validation Hooks

For per-block sanity, drop a tiny `SimConfig.doSim` next to the component.
See `simulation-validation.md` for a full template.
