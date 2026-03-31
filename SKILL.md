---
name: spinal-pro
description: End-to-end SpinalHDL skill for generating, reviewing, explaining, and validating RTL with evidence from official SpinalHDL documentation and mature SpinalHDL codebases such as VexRiscv. Load for SpinalHDL DSL questions, module implementation, interface design, architecture work, bug hunting, and simulation or build validation.
---

# Spinal Pro

## When to Use

Load this skill when the task involves:

- Writing or modifying SpinalHDL RTL
- Reviewing SpinalHDL for semantic bugs, interface bugs, timing-risky structure, or maintainability issues
- Explaining SpinalHDL DSL behavior such as `:=`, `when`, `ClockDomain`, `Stream`, `Flow`, `Bundle`, `Vec`, or `StateMachine`
- Designing parameterized blocks, buses, peripherals, plugin-oriented components, or CPU/SoC structure in SpinalHDL
- Validating SpinalHDL work through local generation, simulation, or repo-native tests
- Using VexRiscv as a pattern source for larger SpinalHDL architecture decisions

## Do Not Use

Do not load this skill for tasks that are purely about general Scala, vendor EDA flows, or unrelated hardware topics unless SpinalHDL semantics or SpinalHDL code are central to the task.

## Workflow

1. Identify the task type: generate, modify, review, explain, retrieve, or design.
2. Inspect the local repository first: target code, build entry points, tests, generators, simulation hooks, and nearby conventions.
3. Open the most relevant bundled reference before reaching for raw corpus material.
4. Prefer official SpinalHDL semantics for language rules and VexRiscv-style examples for architecture and engineering patterns.
5. Produce the smallest evidence-backed answer or change that solves the task.
6. Validate locally when feasible, then return an auditable result with source category, assumptions, and suggested verification steps.

## Evidence Priority

- Use `references/language-semantics.md` for correctness-critical DSL behavior.
- Use `references/review-checklist.md` for review, bug hunting, and regression analysis.
- Use `references/rtl-recipes.md` for common implementation patterns and scaffolding.
- Use `references/architecture-patterns.md` for decomposition, plugins, services, and SoC structure.
- Use `references/simulation-validation.md` before running build, generation, or simulation flows.
- Use `references/vexriscv-casebook.md` to map architecture questions to mature examples.
- Fall back to official SpinalHDL documentation when bundled references are insufficient.
- Use VexRiscv and similar projects as pattern evidence, not as the normative source for language semantics.

## High-Risk Guardrails

Treat the following as high-risk and require explicit evidence plus tighter validation:

- clock and reset strategy, including clock-domain boundaries
- `Stream` / `Flow` / bus handshake correctness
- width inference, truncation, sign extension, literal sizing, and resize behavior
- combinational versus sequential boundaries, latch risk, and combinational loops
- parameter-driven structural changes
- plugin interactions, stage coupling, and hidden side effects
- major CPU or SoC refactors that weaken validation coverage

For high-risk tasks:

- prefer smaller patches over broad rewrites
- state assumptions instead of guessing
- preserve existing interfaces unless evidence supports a change
- avoid cargo-culting from VexRiscv without matching stage timing, reset behavior, and service dependencies
- strengthen validation before declaring the result safe

## Reference Selection

- Open `references/language-semantics.md` for DSL semantics, assignment rules, registers, widths, `ClockDomain`, `Stream`, `Flow`, `Bundle`, `Vec`, and FSM behavior.
- Open `references/review-checklist.md` for code review, bug triage, suspicious constructs, and regression checks.
- Open `references/rtl-recipes.md` for common module patterns, interfaces, buses, FSMs, counters, and register-bank scaffolding.
- Open `references/architecture-patterns.md` for layered decomposition, plugin and service design, parameterization, and SoC integration.
- Open `references/simulation-validation.md` before generation, simulation, or regression work.
- Open `references/vexriscv-casebook.md` when the task needs a concrete VexRiscv file, section, or example to study.

## Output Contract

- Adapt depth to the apparent experience level of the user.
- Separate documented facts from inferences.
- Name the source category used: bundled reference, official SpinalHDL docs, local repo code, or VexRiscv-style example.
- Call out assumptions, unresolved risk, and the next verification step when certainty is incomplete.
