# Review Checklist

Code review, bug triage, and regression-risk analysis for SpinalHDL.

Pair this file with `errors-cheatsheet.md` when looking at concrete error
messages, and `language-semantics.md` for the rule citations.

## Review Order

Walk the diff in this order. The earlier a problem is in this list, the
more expensive it is to find later.

1. **Clock, reset, and domain boundaries** — wrong domain assignments
   cannot be caught by testbench alone.
2. **Interface contracts** — `Stream` / `Flow` / `<>` issues poison every
   downstream consumer.
3. **Assignment completeness and priority** — defaults, last-assignment
   wins, latch risk.
4. **Widths, signedness, resize behavior** — easy to miss in code review,
   easy to test for.
5. **Register lifecycle and state transitions** — including FSM
   completeness.
6. **Parameterization and structural generation** — does the change still
   work for every config variant?
7. **Validation coverage** — does the new behavior have a test path?

## Detection Signals (Yellow Flags)

When you see one of these in a diff, take the time to verify rather than
trust the change:

| Pattern | Worry |
| --- | --- |
| `.resized` or `.resize(...)` across module boundaries | Implicit width change; may truncate silently |
| `.asUInt` / `.asSInt` / `assignFromBits(...)` | Signedness or layout mismatch |
| `allowOverride`, `allowUnsetRegToAvoidLatch`, `noBackendCombMerge` | Compiler check disabled — read the comment justifying it |
| `setAsReg()` | Register created in signal's domain, not the calling area |
| New `ClockDomain(...)` / `ClockingArea(...)` | Verify reset polarity and reset kind |
| `BufferCC`, `PulseCCByToggle`, `StreamCCByToggle`, `FlowCCUnsafeByToggle` | CDC: confirm both sides agree on the protocol |
| `Mem(...)` with `readUnderWrite = writeFirst` | Default emitted Verilog is `readFirst`; need automatic memory blackboxing |
| `\=` on anything that looks like a register | Likely a bug — `\=` does not work on `Reg` |
| `randBoot()` on a register that has consumers | Random-init only, no reset; consumers must not assume zero |
| `valid` depending on `ready` in the same combinational scope | Stream handshake violation |
| `Vec(sig1, sig2)` near a `Reg(Vec(...))` | Check whether the diff confuses aggregation with allocation |
| Plugin or pipeline change in one file with no companion test | Validation gap |

## Clock And Reset

- Verify where each register is created. A register declared outside the
  intended `ClockingArea` keeps the wrong domain even if every assignment
  happens inside the area.
- Verify reset intent. Check `resetKind` (`ASYNC` / `SYNC` / `BOOT`),
  `resetActiveLevel`, and any per-clock-domain `softReset`.
- For CPU/SoC work, review debug-reset separation. Mixing core reset and
  debug reset is a common regression source.
- Flag ad-hoc clock gating unless the codebase already has a known-safe
  primitive. `ClockDomain(gatedClock, ...)` over a logic-derived clock is
  almost always wrong on FPGAs.

Anchor: `language-semantics.md` rule #7;
https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Structuring/clock_domain.html

## Interface Contracts

- For `Bundle with IMasterSlave`, read `asMaster()` before trusting `<>`.
  If `asMaster()` is missing, all downstream `<>` connections quietly
  default to direction-less.
- For `Stream`:
  - `valid` must not depend combinationally on `ready` (see
    [stream.html](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html)).
  - `valid` may only drop the cycle after a `fire`.
  - The payload must not change during a stalled transfer unless the
    interface is documented as "non-locking".
- For `Flow`, verify the consumer truly cannot stall. If the consumer can
  miss events, that needs to be either acceptable by contract or
  upgraded to a `Stream`.
- For memory-mapped blocks, verify read-clear, write-pulse, and any
  synchronous buffering explicitly. `BusSlaveFactory.onRead(addr) { ... }`
  is the canonical place for read side effects.

## Assignment And State

- Look for combinational signals without a default assignment path.
  Heuristic: if a `when` block writes a signal but there is no
  unconditional write before the `when`, that signal needs scrutiny.
- Look for accidental full overwrites that should have been conditional.
  `signal := X` followed by another unconditional `signal := Y` in the
  same scope is an `ASSIGNMENT OVERLAP`.
- Check register update priority. With nested `when`/`elsewhen`,
  remember that the last valid assignment wins, not the first.
- Review uses of `\=`. It rebinds the Scala name; reads later in the
  same scope see the new value. Easy to misuse in helpers that take a
  signal as input.
- For `StateMachine`:
  - every state has a defined `goto` or default behavior
  - state outputs are defaulted outside the state if they must not glitch
  - `onEntry` / `onExit` are deliberately the place for one-shot work
  - `isActive(state)` and `isEntering(state)` are the only safe ways to
    interpret the FSM from outside

## Width, Type, And Arithmetic

- Check every width change around literals, slices, concatenations, and
  arithmetic results. SpinalHDL refuses to widen except for weak-width
  literals.
- Check `UInt` vs `SInt` intent before any cast. Truncation behavior
  differs between the two.
- Check bus payload packing and unpacking. `asBits` packs LSB-first;
  `assignFromBits` follows the same convention.
- Flag any implicit assumption that "Verilog would have widened it" — it
  will not.

## Memory

- For each `Mem` access, identify the port type and the address signal
  domain. Multi-port `Mem`s require careful read-under-write semantics.
- A `readSync(addr, enable = e)` inside a `when(cond)` ignores `cond` —
  fold the condition into `enable` explicitly.
- For `readUnderWrite = writeFirst`, automatic memory blackboxing must
  be enabled via `SpinalConfig(...).addStandardMemBlackboxing(...)`,
  otherwise the generated Verilog will not match the requested semantics.

## Parameterization And Structure

- Check `generate`, `if`, `for` loops, conditional bundle members, and
  plugin lists for structural mismatches across configs.
- Optional features must still leave all required signals driven.
  Common failure: `if (cfg.withExtra) { ... } else { /* nothing */ }`
  forgets to drive `out` ports when `cfg.withExtra` is false.
- Configuration changes that touch stage timing, handshake latency, or
  hazard expectations need a paired test or explicit reasoning in the
  commit message.

## Validation Coverage

- For every behavior change, identify the test that would catch its
  regression. If there isn't one, say so in the review.
- For high-risk areas (clock, reset, handshake, widths), require either
  a focused `doSim` or a written justification of why the existing
  regression suffices.
- For pipeline / CPU refactors that weaken or remove validation, push
  back hard.

## Review Output Pattern

When writing review comments:

- Lead with the highest-risk finding.
- Name the violated contract: handshake contract, width contract,
  reset-domain contract, plugin-stage contract, bus contract.
- Cite the supporting source category. Suggested tags:
  - `[Doc]` — pinned by an official SpinalDoc rule
  - `[Repo]` — derived from the repository's own conventions
  - `[VexRiscv-pattern]` — derived from a mature SpinalHDL codebase
  - `[Inference]` — your interpretation; flag remaining uncertainty
- State the finding's status: confirmed bug, strong suspicion, or
  validation gap.
- Suggest the smallest concrete fix, not a rewrite.

## Useful Anchors

| Topic | URL |
| --- | --- |
| Semantic rules | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Semantic/index.html |
| Design-error catalog | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/index.html |
| FSM | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/fsm.html |
| Stream | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html |
| Flow | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/flow.html |
| `BusSlaveFactory` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/bus_slave_factory.html |
| Plugin library | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/Misc/service_plugin.html |
| Pipeline library | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/Pipeline/introduction.html |
| GCD peripheral (small worked example) | https://github.com/SpinalHDL/VexRiscv/tree/master/doc/gcdPeripheral |
| VexRiscv plugin configs | https://github.com/SpinalHDL/VexRiscv/tree/master/src/main/scala/vexriscv/demo |
