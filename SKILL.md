---
name: spinal-pro
description: End-to-end SpinalHDL skill for generating, reviewing, explaining, and validating RTL (Scala HDL). Covers DSL semantics (`:=`, `<>`, `Reg`, `Stream`, `Flow`, `Bundle`, `Vec`, `ClockDomain`, `StateMachine`, `Mem`, `BusSlaveFactory`), pipeline and plugin composition (`spinal.lib.misc.pipeline`, `spinal.lib.misc.plugin`), simulation with `SimConfig`/`doSim`, and architecture patterns drawn from official SpinalHDL docs and VexRiscv. Load for SpinalHDL DSL questions, module implementation, interface design, SoC integration, bug hunting, error-message decoding, and simulation or build validation.
---

# Spinal Pro

Target SpinalHDL version: 1.10+ (rules and APIs validated against the
`master` branch of SpinalDoc-RTD).

## When To Use

Load this skill when the task involves:

- writing or modifying SpinalHDL RTL
- reviewing SpinalHDL for semantic bugs, interface bugs, timing-risky
  structure, or maintainability issues
- explaining SpinalHDL DSL behavior (`:=`, `\=`, `<>`, `when`, `switch`,
  `ClockDomain`, `Stream`, `Flow`, `Bundle`, `Vec`, `StateMachine`, `Mem`)
- decoding SpinalHDL elaboration errors (`WIDTH MISMATCH`, `LATCH
  DETECTED`, `CLOCK CROSSING VIOLATION`, …)
- designing parameterized blocks, buses, peripherals, plugin-oriented
  components, or CPU/SoC structure
- validating SpinalHDL work through `SimConfig` / `doSim`, focused
  simulation, or repo-native tests
- using VexRiscv as a pattern source for larger architecture decisions

## Do Not Use

Do not load this skill for tasks that are purely about general Scala,
vendor EDA flows, RTL in non-SpinalHDL languages, or unrelated hardware
topics — unless SpinalHDL semantics or SpinalHDL code is central to the
task.

## Workflow

1. **Identify the task type**: generate, modify, review, explain,
   retrieve, or design.
2. **Open the relevant references** for the task type (see table below).
3. **Inspect the local repository** *if* the task touches code:
   target file(s), build entry points, tests, generators, simulation
   hooks, and nearby conventions. Skip this step for pure
   explanation/retrieval tasks.
4. **Use evidence in priority order**: official SpinalHDL semantics for
   language rules, project-local conventions for style, VexRiscv-style
   examples for architecture and engineering patterns.
5. **Produce the smallest evidence-backed answer or change** that solves
   the task.
6. **Validate locally** when feasible (elaboration → focused sim → repo
   regression).
7. **Return an auditable result** with source tags, assumptions, and the
   next verification step.

## Reference Selection

Open the most relevant file first. If the task crosses categories, open
all that apply.

| Task type | Open first | Open also |
| --- | --- | --- |
| Explain a DSL construct or rule | `references/language-semantics.md` | `references/errors-cheatsheet.md` for related errors |
| Decode an elaboration error message | `references/errors-cheatsheet.md` | `references/language-semantics.md` for the rule citation |
| Write a new module or peripheral | `references/rtl-recipes.md` | `references/language-semantics.md` for tricky rules |
| Add a register bank or memory-mapped block | `references/rtl-recipes.md` (BusSlaveFactory + Mem) | `references/architecture-patterns.md` (SoC integration) |
| Build or modify a pipeline | `references/architecture-patterns.md` (Pipeline library) | `references/vexriscv-casebook.md` only if studying VexRiscv |
| Add or restructure a plugin / service | `references/architecture-patterns.md` (Plugin composition) | `references/vexriscv-casebook.md` for stage interactions |
| Code review, bug triage | `references/review-checklist.md` | `references/errors-cheatsheet.md`, `references/language-semantics.md` |
| Write or extend a testbench | `references/simulation-validation.md` | `references/rtl-recipes.md` for the DUT pattern |
| Study mature CPU/SoC patterns | `references/vexriscv-casebook.md` | `references/architecture-patterns.md` |

If a referenced local corpus (`SpinalDoc-RTD/`, `VexRiscv/`) is not
present in the environment, fall back to the URL listed next to each
anchor (`https://spinalhdl.github.io/SpinalDoc-RTD/master/...` for
docs, `https://github.com/SpinalHDL/...` for VexRiscv). If both the
local path and the URL are unreachable, say so in the response — do not
paraphrase from memory.

## High-Risk Guardrails

Treat the following as high-risk and require explicit evidence plus
tighter validation:

- clock and reset strategy, including clock-domain boundaries
- `Stream` / `Flow` / bus handshake correctness
- width inference, truncation, sign extension, literal sizing, resize behavior
- combinational vs sequential boundaries, latch risk, combinational loops
- parameter-driven structural changes
- plugin interactions, stage coupling, hidden side effects
- major CPU or SoC refactors that weaken validation coverage

Detection triggers — when any of these appear in the diff or question,
escalate to high-risk mode:

- `.resized`, `.resize(...)` across module boundaries
- `.asUInt` / `.asSInt` / `assignFromBits(...)`
- `allowOverride`, `allowUnsetRegToAvoidLatch`, comb-loop suppression
- `setAsReg()`, `randBoot()`
- new `ClockDomain(...)` or `ClockingArea(...)`
- `BufferCC`, `PulseCCByToggle`, `StreamCCByToggle`, `FlowCCUnsafeByToggle`
- `Mem(...)` with `readUnderWrite = writeFirst`
- `\=` on anything that might be a register

For high-risk tasks:

- prefer smaller patches over broad rewrites
- state assumptions instead of guessing
- preserve existing interfaces unless evidence supports a change
- avoid cargo-culting from VexRiscv without matching stage timing, reset
  behavior, and service dependencies
- strengthen validation before declaring the result safe

## Output Contract

Annotate every load-bearing claim with one of these source tags so a
reviewer can audit at a glance:

- `[Doc]` — pinned by an official SpinalDoc rule (cite the anchor or URL)
- `[Repo]` — derived from the current repository's code or conventions
- `[VexRiscv-pattern]` — derived from a mature SpinalHDL codebase
- `[Inference]` — your interpretation; flag remaining uncertainty

Depth heuristic — pick one of three modes based on what the user
demonstrates:

- **Direct**: experienced user, terse question → short answer with code
  and one citation; skip background.
- **Guided**: mixed or unclear → answer + a one-line "why this matters"
  + one citation.
- **Teaching**: clear beginner cue (asks what `:=` is, what `Bundle`
  means, etc.) → answer + minimal example + the rule citation, but stop
  short of unrelated background.

End any non-trivial output with:

- a one-line statement of which source category dominated the answer
- unresolved risks or assumptions, if any
- the next verification step (a sim command, a file to inspect, a
  reviewer to flag) when certainty is incomplete

## Recipes Directory

```
references/
├── language-semantics.md     # DSL rules, operators, width, registers, when/switch, Bundle/Vec
├── rtl-recipes.md            # Module skeletons, Stream toolkit, Mem, BusSlaveFactory, FSM, CDC, SpinalConfig
├── architecture-patterns.md  # Decomposition, Pipeline library, plugin composition, SoC patterns
├── review-checklist.md       # Review order, detection signals, output pattern
├── errors-cheatsheet.md      # One-screen error → cause → fix lookups
├── simulation-validation.md  # SpinalSim API, testbench templates, validation ladder
└── vexriscv-casebook.md      # VexRiscv patterns with preconditions and anti-patterns
```

## Evidence Sources

| Source | URL |
| --- | --- |
| SpinalHDL documentation (master) | https://spinalhdl.github.io/SpinalDoc-RTD/master/index.html |
| SpinalHDL source | https://github.com/SpinalHDL/SpinalHDL |
| SpinalHDL documentation source | https://github.com/SpinalHDL/SpinalDoc-RTD |
| VexRiscv | https://github.com/SpinalHDL/VexRiscv |
| NaxRiscv (newer plugin/fiber style) | https://github.com/SpinalHDL/NaxRiscv |
