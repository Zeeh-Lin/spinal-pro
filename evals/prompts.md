# Evaluation Prompts

A small forward-test set for the `spinal-pro` skill. Use these prompts to
sanity-check that loading the skill produces auditable, evidence-backed
answers — and to catch regressions when the skill or its references
change.

Each prompt is paired with a rubric: the signals that should appear in a
good answer, and the failure modes to watch for.

## How To Use

1. Start a fresh session with the skill loaded.
2. Paste one prompt at a time.
3. Score the response against the rubric (Pass / Partial / Fail).
4. Record failures and update the responsible reference file.

A Pass requires every "Must" bullet to be present. A Partial answer has
at least one "Must" bullet missing. A Fail makes a factually wrong claim
about SpinalHDL semantics.

---

## Tier 1 — Explanation

### P1.1 Assignment operators

**Prompt.** What is the difference between `:=`, `\=`, and `<>` in
SpinalHDL? When would I reach for each?

**Must.**
- `:=` is standard assignment, last-valid-wins under `when`/`switch`.
- `\=` is immediate in-place update for combinational signals; does not
  work on registers.
- `<>` is direction-driven structural connect using `IMasterSlave`.
- Cite either `language-semantics.md` or the `Semantic/assignments.html`
  URL.

**Watch for.**
- Claiming `\=` is a register update operator.
- Claiming `<>` works on any pair of signals without direction inference.

### P1.2 Why latches happen

**Prompt.** I am getting `LATCH DETECTED from the combinatorial signal
toplevel/a`. What does this mean and what is the smallest fix?

**Must.**
- Identifies missing default assignment as the cause.
- Shows the fix: assign a default before the conditional branch.
- Mentions the alternative: make it a `Reg` if state retention was
  intended.
- Cites `errors-cheatsheet.md` or the `latch_detected.html` URL.

**Watch for.**
- Suggesting `--allowLatch` or other suppressions as the first fix.

### P1.3 Stream handshake rules

**Prompt.** Summarize the rules my `Stream` producer must follow so
SpinalHDL accepts it and downstream consumers behave.

**Must.**
- Transfer happens only when `valid && ready`.
- `valid` may only drop the cycle after a `fire`.
- Payload must not change during a stalled transfer (note exceptions
  for documented non-locking interfaces).
- `valid` must not depend combinationally on `ready`.
- Cites `language-semantics.md` rule 8 or the `stream.html` URL.

**Watch for.**
- Forgetting the `valid` → `ready` direction.
- Adding fabricated additional rules.

---

## Tier 2 — Code Generation

### P2.1 Stream FIFO bridge

**Prompt.** Write a small SpinalHDL `Component` named `StreamBridge` that
takes a `slave Stream(UInt(16 bits))` and a `master Stream(UInt(16 bits))`
and inserts a 4-deep FIFO with one cycle of forward latency. Show the
imports.

**Must.**
- Imports `spinal.core._` and `spinal.lib._`.
- Uses `io.up.queue(4)` or `StreamFifo(...)` with `m2sPipe()` /
  equivalent.
- IO matches the names and directions requested.
- Provides a `SpinalVerilog(new StreamBridge)` or equivalent
  generation entry point (or notes how to generate).

**Watch for.**
- Hand-rolling the FIFO when a stock helper would do.
- Wiring `valid` and `ready` manually with bugs.

### P2.2 Edge-detected event

**Prompt.** Implement an event reporter that emits a `Flow(NoData())`
pulse on each rising edge of a `Bool` input. Default the input to `False`
on reset and account for the first cycle.

**Must.**
- Uses `RegNext(input) init(False)` or equivalent.
- Drives `flow.valid := input && !prev`.
- Notes that `Flow` cannot stall, so the consumer must be ready every
  cycle.

### P2.3 APB3 register bank

**Prompt.** Build an APB3 peripheral that exposes a single 32-bit
read/write register at address `0x00` and a 32-bit read-only "version"
register at `0x04` (value `0x1234_5678`).

**Must.**
- Imports `spinal.lib.bus.amba3.apb._`.
- Uses `Apb3SlaveFactory`.
- `readAndWrite(reg, 0x00)` for the writable register.
- `read(U("32'h12345678"), 0x04)` for the version register.
- Provides a sensible `Apb3Config` argument or notes it.

**Watch for.**
- Hand-rolling APB address decoding.
- Inventing a `BusSlaveFactory` method that does not exist.

### P2.4 Synchronous FIFO with backpressure scoreboard

**Prompt.** Write a `SimConfig.doSim` testbench for `StreamBridge` from
P2.1 that uses `StreamDriver`, `StreamReadyRandomizer`, and a
`ScoreboardInOrder` to confirm in-order delivery under random
backpressure for 1000 cycles.

**Must.**
- Uses `spinal.lib.sim._` helpers.
- Forks the clock with `forkStimulus(period)`.
- Sets a `SimTimeout`.
- Calls `scoreboard.checkEmptyness()` at the end.

---

## Tier 3 — Review / Bug Hunt

### P3.1 Bad Stream

**Prompt.** Review this code:

```scala
class BadStream extends Component {
  val io = new Bundle {
    val in  = slave  Stream(UInt(8 bits))
    val out = master Stream(UInt(8 bits))
  }

  io.out.valid := io.in.valid && io.in.ready
  io.in.ready  := io.out.ready
  io.out.payload := io.in.payload
}
```

**Must.**
- Identify the `valid := valid && ready` combinational dependency of
  `valid` on `ready` as a Stream-contract violation.
- Suggest the canonical fix (`io.out << io.in` or `io.out <-< io.in`).
- Tag the finding (`[Doc]` Stream contract).

### P3.2 CDC without synchronizer

**Prompt.** A junior engineer wrote:

```scala
val regA = clkA(Reg(UInt(8 bits)) init(0))
val regB = clkB(Reg(UInt(8 bits)) init(0))
regB := regA + 1
```

What is wrong and what is the fix?

**Must.**
- Diagnose `CLOCK CROSSING VIOLATION` (or equivalent).
- Recommend `BufferCC` only for the level-stable case; otherwise
  `StreamCCByToggle` / `StreamFifoCC` for buses, `PulseCCByToggle` for
  single pulses.
- Note that `regA + 1` cannot be moved across a 2FF synchronizer
  bit-by-bit safely — needs a bus-level CDC primitive.

### P3.3 Latch via missing default

**Prompt.** Find the bug:

```scala
val a = UInt(8 bits)
when(io.sel) {
  a := 0x42
}
io.out := a
```

**Must.**
- Diagnose latch / `LATCH DETECTED`.
- Show the fix (default assignment before `when`).
- Cite the rule from `language-semantics.md` or the doc URL.

---

## Tier 4 — Architecture

### P4.1 Pipeline library vs hand wiring

**Prompt.** I have a 3-stage compute pipeline (input → multiply → add).
Should I use `spinal.lib.misc.pipeline` or wire registers manually?
Justify in 3-5 sentences.

**Must.**
- Recommend the pipeline library when stage count or signals may change.
- Note that hand wiring is fine for tiny fixed-shape pipelines.
- Mention that the library generates the `valid`/`ready` arbitration
  automatically.
- Cite `architecture-patterns.md` Pipeline section or the
  `Pipeline/introduction.html` URL.

### P4.2 When to reach for plugins

**Prompt.** When does a SpinalHDL component justify using
`spinal.lib.misc.plugin` + `PluginHost` over plain subcomponents?

**Must.**
- Mentions "extensibility" / "feature swap" as the key driver.
- Notes the two-phase elaboration (`during setup` / `during build`).
- Calls out the cost: harder to read, harder to debug.
- Cites `architecture-patterns.md` Plugin section or the
  `service_plugin.html` URL.

### P4.3 Cargo-cult check

**Prompt.** I am porting `BranchPlugin` from VexRiscv into a smaller
custom core. What must I check before assuming it will work?

**Must.**
- Same stage timing assumptions.
- Same reset and exception contracts.
- Service dependencies (`HazardService`, `DecoderService`, etc.).
- Frontend redirect / prediction semantics.
- Cites the `vexriscv-casebook.md` Do-Not-Cargo-Cult checklist.

---

## Tier 5 — Validation

### P5.1 Choosing a validation tier

**Prompt.** I changed the width of a counter from 16 to 17 bits to fix a
rollover bug. What is the minimum validation I should run?

**Must.**
- Suggest elaboration check first (`generateVerilog`).
- Add a focused sim that exercises the rollover boundary at the new
  width.
- Skip large regression unless other code depends on the old width.
- Mention `language-semantics.md` width rules in passing.

### P5.2 Detecting a regression in a long sim

**Prompt.** A simulation that previously passed now fails after 50000
cycles with `SimTimeout`. What do I check?

**Must.**
- Inspect the wave (`SimConfig.withFstWave` if not already on).
- Check whether `forkStimulus` is still running the clock.
- Look for a stalled `Stream` (no `fire` for many cycles).
- Consider `simSuccess` / `simFailure` race conditions in test forks.
- Cites `simulation-validation.md`.

---

## Notes For Future Maintainers

- Keep the prompt list short (under ~20). The point is regression
  signal, not coverage.
- Whenever a real-world failure surfaces, add a prompt that would have
  caught it.
- A prompt that passes consistently for three updates can be archived;
  keep prompts that exercise current pain points.
