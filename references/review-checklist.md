# Review Checklist

Use this reference for SpinalHDL code review, bug triage, and regression-risk analysis.

## Review Order

1. Check clock, reset, and domain boundaries.
2. Check interface contracts.
3. Check assignment completeness and priority.
4. Check widths, signedness, and resize behavior.
5. Check register lifecycle and state transitions.
6. Check parameterization and structural generation.
7. Check validation coverage.

## Clock And Reset

- Verify where each register is created.
  A register created outside the intended `ClockingArea` keeps the wrong domain even if later assignments happen inside the area.
- Verify reset intent.
  Review reset kind, polarity, and debug-versus-core reset separation for CPU or SoC work.
- Flag ad hoc clock gating unless the repository already uses a known-safe primitive or pattern.
- For CPU-like designs, review debug-domain exceptions explicitly.
  VexRiscv documents separate debug reset behavior in `VexRiscv/README.md` and implements multi-domain reset composition in `VexRiscv/src/main/scala/vexriscv/demo/Murax.scala`.

## Interface Contracts

- For `Bundle with IMasterSlave`, inspect `asMaster()` before trusting `<>`.
- For `Stream`, verify:
  `valid` persistence, no combinational dependency on `ready`, and correct `fire` behavior.
- For `Flow`, verify that the consumer truly cannot stall.
- For memory-mapped blocks, verify read-clear, write-pulse, and sync buffer behavior explicitly.

## Assignment And State

- Look for combinational signals without a default assignment path.
- Look for accidental full overwrites that should have been conditional.
- Check register updates for intended priority order.
- Review uses of `\=` carefully.
  It changes local read-after-write semantics and is easy to misuse.
- Check `StateMachine` logic for:
  missing transitions, invalid completion conditions, and state outputs that should be defaulted outside the active state.

## Width, Type, And Arithmetic

- Check every width change around literals, slices, concatenations, and arithmetic results.
- Check `UInt` versus `SInt` intent before resizing or casting.
- Check bus payload packing and unpacking with `asBits` or `assignFromBits`.
- Flag implicit assumptions about automatic widening outside weak literals.

## Parameterization And Structure

- Check `generate`, `ifGen`, conditional bundle members, and plugin lists for structural mismatches.
- Check that optional features still leave all required signals driven.
- Check that configuration changes do not silently change stage timing, handshake latency, or hazard expectations.

## VexRiscv-Specific Smells

- Result stage and hazard flags disagree.
  Example: a plugin writes late but advertises execute-stage bypassability.
- Frontend latency and regfile timing disagree.
  The VexRiscv README calls out the `Missing inserts : INSTRUCTION_ANTICIPATE` failure mode.
- Branch timing changes without re-checking frontend prediction or penalty assumptions.
- Debug features change reset behavior without updating surrounding logic.
- A pattern is copied from a different plugin configuration without matching its services or pipeline stages.

## Useful Anchors

- Semantic rules:
  `SpinalDoc-RTD/source/SpinalHDL/Semantic/*.rst`
- Design-error catalog:
  `SpinalDoc-RTD/source/SpinalHDL/Design errors/*.rst`
- FSM and handshake docs:
  `SpinalDoc-RTD/source/SpinalHDL/Libraries/fsm.rst`
  `SpinalDoc-RTD/source/SpinalHDL/Libraries/stream.rst`
  `SpinalDoc-RTD/source/SpinalHDL/Libraries/flow.rst`
- Simple control-plus-datapath example:
  `VexRiscv/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/*.scala`
- Plugin timing/config examples:
  `VexRiscv/README.md`
  `VexRiscv/src/main/scala/vexriscv/demo/*.scala`

## Review Output Pattern

- Report the highest-risk findings first.
- Name the violated contract.
  Examples: handshake contract, width contract, reset-domain contract, plugin-stage contract.
- Cite the supporting source category.
- State whether the issue is a confirmed bug, a strong suspicion, or a validation gap.
