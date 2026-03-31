# Language Semantics

Use this reference for correctness-critical SpinalHDL behavior.

## Quick Map

- Assignments and width rules:
  `SpinalDoc-RTD/source/SpinalHDL/Semantic/assignments.rst`
- Rule model, concurrency, and last-assignment-wins:
  `SpinalDoc-RTD/source/SpinalHDL/Semantic/rules.rst`
- `when`, `switch`, `Mux`, and exhaustiveness:
  `SpinalDoc-RTD/source/SpinalHDL/Semantic/when_switch.rst`
- Registers and reset behavior:
  `SpinalDoc-RTD/source/SpinalHDL/Sequential logic/registers.rst`
- Clock domains:
  `SpinalDoc-RTD/source/SpinalHDL/Structuring/clock_domain.rst`
- `Bundle` and `IMasterSlave`:
  `SpinalDoc-RTD/source/SpinalHDL/Data types/bundle.rst`
- `Vec`:
  `SpinalDoc-RTD/source/SpinalHDL/Data types/Vec.rst`
- `Stream`:
  `SpinalDoc-RTD/source/SpinalHDL/Libraries/stream.rst`
- `Flow`:
  `SpinalDoc-RTD/source/SpinalHDL/Libraries/flow.rst`
- FSM library:
  `SpinalDoc-RTD/source/SpinalHDL/Libraries/fsm.rst`
- Error references:
  `SpinalDoc-RTD/source/SpinalHDL/Design errors/*.rst`

If the local corpus is absent, use the same chapter names on the official SpinalHDL documentation site.

## Normative Rules

- Treat signal nature as declaration-time semantics.
  `UInt(...)`, `Bits(...)`, `Bool()`, `Bundle`, and `Vec` create combinational signals unless wrapped by `Reg(...)` or a register helper.
- Treat `:=` as standard assignment with "last valid assignment wins".
  Multiple valid assignments in prioritized control flow are expected.
  A full overwrite from the same scope is an assignment-overlap error unless explicitly allowed.
- Treat `\=` as immediate in-place combinational update only.
  Do not use it on registers.
  Review it carefully because it changes read-after-write behavior inside the same elaborated scope.
- Treat `<>` as direction-driven structural connection.
  Verify `IMasterSlave.asMaster()` before trusting bundle auto-wiring.
- Use explicit width adaptation.
  Prefer `.resized`, `.resize(newWidth)`, or `.resizeLeft(newWidth)` when widths differ.
  Do not assume implicit narrowing.
- Remember the one automatic exception.
  Weak-width literals such as `U(3)` can widen automatically when assigned to a larger target, but this does not justify relying on implicit conversions elsewhere.
- Give combinational outputs a default path.
  Missing coverage in `when`, `switch`, or `mux` logic causes latch detection.
- Treat registers as state holders with self-retention.
  If no assignment fires, the register keeps its value.
  If a register impacts the design and never gets assigned, SpinalHDL reports an unassigned-register error.
- Create registers inside the intended `ClockingArea`.
  Clock-domain capture happens when the register is created, not where later assignments appear.
- Use `Flow` only when the sink cannot backpressure.
  `Flow` has `valid` and `payload` only.
- Enforce `Stream` handshake rules strictly.
  A transfer happens only when `valid && ready`.
  `valid` must not depend combinatorially on `ready`.
  Prefer no dependency at all from `valid` to `ready`.
- Use `Vec(x, y, ...)` with care.
  It creates a vector of references and does not create new hardware.

## High-Risk Semantic Checks

Apply extra scrutiny when the task touches:

- `ClockDomain`, reset polarity, reset kind, or `ClockingArea`
- `Stream` / `Flow` bridges, FIFOs, or custom ready/valid logic
- signed versus unsigned arithmetic, literal sizing, and explicit casts
- `setAsReg()`, especially when used across areas
- nested `when` or `switch` trees without obvious defaults
- `mux` or `muxList` logic that might not be exhaustive
- manual overrides using `allowOverride` or comb-loop suppression helpers

## Error-Driven Reading Guide

- Width mismatch:
  open `Design errors/width_mismatch.rst`
- Latch detected:
  open `Design errors/latch_detected.rst`
- Combinational loop:
  open `Design errors/combinatorial_loop.rst`
- Assignment overlap:
  open `Design errors/assignment_overlap.rst`
- No driver:
  open `Design errors/no_driver_on.rst`
- Unassigned register:
  open `Design errors/unassigned_register.rst`

## Practical Interpretation

- Use official docs to answer "what does this mean in SpinalHDL?"
- Use local project code to answer "how is this repo already doing it?"
- Use VexRiscv only after the first two are clear, and only to learn patterns, staging, or integration style.
