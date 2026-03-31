# Simulation And Validation

Use this reference before declaring SpinalHDL work complete.

## Validation Ladder

Run the lightest checks that can still falsify the change:

1. Elaboration or generation check
2. Focused block simulation
3. Subsystem integration check
4. Wider regression or repository-native test suite

Escalate when the task touches high-risk semantics such as reset, handshake, widths, or pipeline timing.

## Local Discovery Order

- Inspect `build.sbt`, `build.sc`, `mill` files, and Makefiles first.
- Inspect `src/test`, `doc/*Sim.scala`, or repo-specific simulation harnesses next.
- Prefer the repository's existing entry points over inventing a new one-off flow.

Useful local anchors in the provided corpus:

- `VexRiscv/build.sbt`
- `VexRiscv/build.sc`
- `VexRiscv/README.md`
- `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDTopSim.scala`
- `VexRiscv/src/test/scala/vexriscv/*.scala`
- `VexRiscv/src/test/cpp/**`

## Common Commands

Use commands like these when the repository exposes them:

- Generate RTL from a Scala main:
  `sbt "runMain vexriscv.demo.GenFull"`
- Run focused Scala tests:
  `sbt "testOnly vexriscv.TestIndividualFeatures"`
- Run broader Scala tests:
  `sbt test`
- Run simulation mains if the repo wires them into the build:
  `sbt "runMain <package>.YourSim"`

The VexRiscv README also documents regression environment variables such as:

- `VEXRISCV_REGRESSION_SEED`
- `VEXRISCV_REGRESSION_TEST_ID`

Use them to reproduce or narrow failures instead of rerunning everything blindly.

## What To Check

- Reset behavior:
  initial values, reset release, repeated reset, and debug-reset interaction when relevant
- Handshake behavior:
  sustained backpressure, bursts, stalls, and ready/valid persistence
- Width-sensitive behavior:
  boundary literals, truncation, sign behavior, packed fields, and address alignment
- State behavior:
  idle, first transaction, repeated transaction, and completion conditions
- Parameter behavior:
  smallest supported config, representative default config, and one stress config

## Evidence Patterns

- Prefer a software model or scoreboard when the function is easy to compute.
- Prefer random plus constrained-corner stimulus for arithmetic or protocol blocks.
- Prefer existing regression hooks for CPU or SoC changes.
- For review-only tasks, state which validation should be run even if it cannot be executed in the current environment.

## Corpus Anchors

- Official simulation overview:
  `SpinalDoc-RTD/source/SpinalHDL/Simulation/index.rst`
- Example algorithmic block simulation:
  `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDTopSim.scala`
- Regression entry points:
  `VexRiscv/README.md`

## Release Criteria

Treat the work as ready only when the most relevant local check has passed, or when the result clearly states:

- what was not validated
- why it was not validated
- which exact command or workflow should be run next
