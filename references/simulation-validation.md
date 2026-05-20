# Simulation And Validation

Use this reference before declaring SpinalHDL work complete. It covers the
native SpinalSim API, common testbench templates, and what to escalate to
when the change is risky.

Target version: SpinalHDL 1.10+ with Verilator as the default backend.

## Validation Ladder

Run the lightest check that can still falsify the change:

1. **Elaboration / generation** — `SpinalConfig(...).generateVerilog(...)` runs all SpinalHDL design-rule checks. Always free, do it first.
2. **Focused block simulation** — `SimConfig.compile(...).doSim { dut => ... }` with a minimal stimulus and a software model where one exists.
3. **Subsystem integration check** — run the next level up that the repo provides (CPU + bus + memory, peripheral + driver model, …).
4. **Repo-native regression** — `sbt test`, `mill <module>.test`, or the project's custom regression script.
5. **Formal** (optional, high-risk only) — see the formal anchors below.

Escalate when the task touches `ClockDomain`, reset, `Stream`/`Flow`
handshakes, `Mem` read-under-write, widths around signed/unsigned, or any
pipeline timing change.

## Local Discovery Order

Before writing any new test, look for what is already wired up:

```bash
ls build.sbt build.sc mill Makefile 2>/dev/null
find . -path '*/src/test*' -name '*.scala'
find . -name '*Sim.scala' -not -path '*/target/*'
```

Reuse the repository's existing testbench entry points when they exist;
inventing a parallel `doSim` for a project that already has one fragments
maintenance.

## Minimal SpinalSim Template

```scala
import spinal.core._
import spinal.core.sim._

class Adder(width: Int) extends Component {
  val io = new Bundle {
    val a, b = in  UInt(width bits)
    val sum  = out UInt(width + 1 bits)
  }
  io.sum := io.a.resize(width + 1) + io.b.resize(width + 1)
}

object AdderSim extends App {
  SimConfig.withWave.compile(new Adder(8)).doSim { dut =>
    for (a <- 0 until 256; b <- 0 until 256 by 17) {
      dut.io.a #= a
      dut.io.b #= b
      sleep(1)
      val got = dut.io.sum.toBigInt
      val exp = BigInt(a) + BigInt(b)
      assert(got == exp, s"$a + $b expected $exp got $got")
    }
  }
}
```

Key APIs:

| Need | Idiom |
| --- | --- |
| Drive an input | `dut.io.foo #= value` |
| Read an output | `dut.io.foo.toInt` / `.toBigInt` / `.toBoolean` |
| Wait one delta cycle | `sleep(1)` |
| Drive a clock | `dut.clockDomain.forkStimulus(period = 10)` |
| Wait for next rising edge sample | `dut.clockDomain.waitSampling()` |
| Wait until a condition is sampled true | `dut.clockDomain.waitSamplingWhere(cond)` |
| Reset the DUT | `dut.clockDomain.assertReset(); sleep(p); dut.clockDomain.deassertReset()` |
| Set a hard timeout | `SimTimeout(maxCycles * period)` |
| End the sim from a fork | `simSuccess()` / `simFailure(msg)` |

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Simulation/bootstraps.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Simulation/bootstraps.html

## Clocked Testbench Template

```scala
object CounterSim extends App {
  SimConfig.withFstWave.compile(new Counter).doSim { dut =>
    SimTimeout(1000 * 10)
    dut.clockDomain.forkStimulus(period = 10)

    // Reset for a few cycles
    dut.clockDomain.assertReset()
    dut.clockDomain.waitSampling(3)
    dut.clockDomain.deassertReset()

    // Stimulus loop
    for (_ <- 0 until 20) {
      dut.clockDomain.waitSampling()
      assert(dut.io.value.toInt < 256)
    }
  }
}
```

`forkStimulus` starts a forked thread that toggles the clock — without it,
no register ever updates.

## Stream / Flow Driving

`spinal.lib.sim` ships ready-made helpers:

```scala
import spinal.lib.sim._

StreamDriver(dut.io.cmd, dut.clockDomain) { payload =>
  payload #= scala.util.Random.nextInt(256)
  true  // return true to keep producing transactions
}

StreamReadyRandomizer(dut.io.rsp, dut.clockDomain)

val scoreboard = ScoreboardInOrder[BigInt]()
StreamMonitor(dut.io.cmd, dut.clockDomain) { p => scoreboard.pushRef(p.toBigInt) }
StreamMonitor(dut.io.rsp, dut.clockDomain) { p => scoreboard.pushDut(p.toBigInt) }

dut.clockDomain.waitSampling(1000)
scoreboard.checkEmptyness()
```

These cover the most common backpressure patterns; combine with
`Random.nextInt(...)` for stress.

## SpinalConfig Inside Simulation

`SimConfig` accepts a `SpinalConfig` override when you need the generated
RTL to match a specific target setting:

```scala
val spinalConfig = SpinalConfig(
  defaultClockDomainFrequency = FixedFrequency(100 MHz),
  defaultConfigForClockDomains = ClockDomainConfig(
    resetKind        = ASYNC,
    resetActiveLevel = HIGH
  )
)

SimConfig
  .withConfig(spinalConfig)
  .withFstWave
  .workspacePath("simWorkspace")
  .compile(new Cpu)
  .doSim("smoke") { dut => /* ... */ }
```

Wave output:
- `withWave` / `withVcdWave` / `withFstWave` choose the dump format.
- Default workspace: `simWorkspace/<TopName>/<TestName>/`. Override with
  `SimConfig.workspacePath(...)` or the `SPINALSIM_WORKSPACE` env var.

## What To Check Per Task Type

| Change | Minimum stimulus to exercise |
| --- | --- |
| New combinational block | Random inputs vs software model; corner literals at width boundaries |
| New register / FSM | Reset → idle → first transaction → repeated transaction → reset again |
| Stream / Flow plumbing | Sustained backpressure, random `ready` gaps, isolated single transactions, bursts |
| Memory-mapped peripheral | Per-register read, write, read-after-write, write-after-write, side effects (clear-on-read) |
| Clock-domain crossing | Asymmetric clock ratios, slow→fast and fast→slow, sustained traffic, idle periods |
| Pipeline-stage refactor | Identical golden-vs-modified trace under the same stimulus |

## Common Commands

The commands below are conventional; adapt them to the actual repo.

```bash
# Generate Verilog from a Scala main
sbt "runMain mypkg.MyGen"
mill myproj.runMain mypkg.MyGen

# Focused Scala simulation
sbt "runMain mypkg.MySim"

# Run unit tests
sbt test
sbt "testOnly mypkg.MySpec"
mill myproj.test

# Reproduce a specific failing case
SPINALSIM_WORKSPACE=./tmp sbt "runMain mypkg.MySim --seed 1234"
```

For VexRiscv-style projects, the README documents environment variables
such as `VEXRISCV_REGRESSION_SEED` and `VEXRISCV_REGRESSION_TEST_ID` to
narrow regression scope before rerunning everything.

## Memory Manipulation In Sim

```scala
val mem = dut.fetcher.mem      // requires .simPublic on the Mem
mem.setBigInt(0, BigInt("0042", 16))
val v   = mem.getBigInt(0)
```

Mark RAMs you want to poke with `Mem.fill(...).simPublic` (or `.simPublic`
on the existing `Mem`). Without that, the simulator cannot reach the
internal storage.

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Simulation/signal.rst` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Simulation/signal.html

## Debugging A Hung Or Timed-Out Simulation

When a sim that previously passed now stalls or hits `SimTimeout`, walk
this checklist in order — most failures fall in the first three.

1. **Confirm the clock is running.** Without `dut.clockDomain.forkStimulus(period)`,
   no register updates. Easy to lose during a refactor.
2. **Capture a wave window.** Re-run with `SimConfig.withFstWave` (or
   `withVcdWave`) and inspect the last 100 cycles in GTKWave / Surfer.
   Wave file lives under `simWorkspace/<TopName>/<TestName>/`. If the sim
   is too long, use `DualSimTracer` to record only the window before
   failure.
3. **Look for a stalled Stream.** Search the wave for any `*_valid` high
   while the matching `*_ready` is low for many cycles. That's almost
   always the proximate cause. Cross-reference against the
   producer/consumer logic.
4. **Check `simSuccess` / `simFailure` placement.** If the testbench uses
   `doSimUntilVoid`, every fork must terminate via `simSuccess()` /
   `simFailure(msg)` or the sim waits forever. A bare `doSim` exits when
   the main thread returns — but a fork can still hang it via `simThread.suspend()`.
5. **Inspect any `waitSamplingWhere(cond)`.** A condition that never
   becomes true freezes the thread. Add a `SimTimeout(...)` so it fails
   loudly rather than silently hanging.
6. **Bisect with a smaller `SimTimeout`.** Halve the timeout and rerun;
   the failure point moves earlier in the wave and is easier to read.
7. **Reproduce with a fixed seed.** Add `doSim("name", seed = 1234)` so
   the next iteration starts from the same state.
8. **Check for FIFO over-fill or arbitration starvation.** If a producer
   never sees `ready`, look at the next consumer's `ready` source — often
   a back-pressure chain from elsewhere.

If none of the above lights up, suspect a generator bug (the elaborated
RTL no longer matches what the testbench thinks it drives) and re-run
`SpinalConfig(...).generateVerilog(...)` to inspect the regenerated
module.

## When To Reach For Formal

SpinalHDL provides a thin wrapper over SymbiYosys for formal verification.
Reach for it when:

- The block has a tight handshake contract you want to prove (no lost
  transactions, no spurious `valid` after `ready` drop).
- A bit-level invariant is easier to express than to test exhaustively.
- An incident postmortem demands a proof rather than coverage.

Anchor: `SpinalDoc-RTD/source/SpinalHDL/Formal verification/` ·
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Formal%20verification/index.html

## Release Criteria

Treat work as ready only when the most relevant local check has passed, or
when the result clearly states:

- which validation was run, with seed and stimulus shape
- which validation could not be run, and why
- which exact command should be run next by a human

Never claim "simulation passes" without naming the test, the seed, and the
duration.
