# spinal-pro

`spinal-pro` is an agent skill for practical SpinalHDL development.

It helps AI coding agents generate, review, explain, and validate
SpinalHDL RTL with a correctness-first workflow. The skill grounds every
load-bearing claim in either the official SpinalHDL documentation or
mature SpinalHDL projects such as VexRiscv, and surfaces an auditable
output format so a human reviewer can trace each answer.

Target SpinalHDL version: 1.10+ (validated against the `master` branch
of SpinalDoc-RTD at the time of writing).

This repository is packaged for OpenSkills, but it can also be used as
a plain `SKILL.md` repository.

## When To Use

Use `spinal-pro` when the task involves:

- writing or modifying SpinalHDL RTL
- reviewing SpinalHDL for semantic bugs, interface bugs, or structural
  risks
- explaining SpinalHDL DSL behavior (`:=`, `\=`, `<>`, `when`, `switch`,
  `ClockDomain`, `Stream`, `Flow`, `Bundle`, `Vec`, `StateMachine`,
  `Mem`)
- decoding SpinalHDL elaboration error messages
- designing parameterized modules, interfaces, buses, peripherals, or
  larger CPU/SoC structures
- validating SpinalHDL work through `SimConfig` / `doSim`, local
  simulation, or repo-native tests

This skill is for practical SpinalHDL work. It is intentionally not a
general Scala or vendor-specific EDA-flow assistant unless SpinalHDL is
central to the task.

## Quick Start

Install from GitHub via OpenSkills:

```bash
npx openskills install Zeeh-Lin/spinal-pro --universal
npx openskills sync
```

### Without OpenSkills

If your agent already understands Anthropic-style skills, it can read
`SKILL.md` plus the files under `references/` directly. No extra
metadata is required.

## Repository Layout

```text
spinal-pro/
├── README.md
├── SKILL.md
├── references/
│   ├── language-semantics.md     # DSL rules, operators, width, registers, when/switch, Bundle/Vec
│   ├── rtl-recipes.md            # Module skeletons, Stream toolkit, Mem, BusSlaveFactory, FSM, CDC, SpinalConfig
│   ├── architecture-patterns.md  # Decomposition, Pipeline library, plugin composition, SoC patterns
│   ├── review-checklist.md       # Review order, detection signals, output pattern
│   ├── errors-cheatsheet.md      # One-screen error → cause → fix lookups
│   ├── simulation-validation.md  # SpinalSim API, testbench templates, validation ladder
│   └── vexriscv-casebook.md      # VexRiscv patterns with preconditions and anti-patterns
└── evals/
    └── prompts.md                # Forward-test prompts and rubrics
```

- `SKILL.md` is the operating contract: trigger criteria, workflow,
  reference selection, high-risk guardrails, and output format.
- `references/*.md` are opened on demand by task type — each contains
  self-contained code snippets and links back to the underlying
  SpinalDoc-RTD anchor (both local path and public URL).
- `evals/prompts.md` is a short regression set for forward-testing the
  skill after edits.

## Design Principles

- **Correctness first.** Cite official SpinalHDL docs for language
  rules; treat VexRiscv as pattern evidence, not as the language spec.
- **Progressive disclosure.** `SKILL.md` stays short and procedural;
  detailed knowledge lives under `references/`.
- **Self-contained references.** Every reference file embeds minimal
  Scala code snippets, so the skill remains useful even when the local
  SpinalDoc-RTD / VexRiscv corpus is not checked out.
- **URL fallback.** When the local corpus is missing, the skill falls
  back to the public URLs listed next to each anchor.
- **Auditable outputs.** Each answer carries source tags (`[Doc]`,
  `[Repo]`, `[VexRiscv-pattern]`, `[Inference]`) so reviewers can
  verify claims at a glance.

## Scope

This skill focuses on:

- SpinalHDL DSL semantics
- practical RTL implementation patterns (Stream / Flow / Bundle / FSM /
  Mem / BusSlaveFactory)
- pipeline and plugin composition (`spinal.lib.misc.pipeline`,
  `spinal.lib.misc.plugin`)
- review guidance and elaboration-error decoding
- simulation and validation with the native `SimConfig` / `doSim` API
- architecture patterns from real SpinalHDL codebases

Non-goals:

- general Scala expertise
- vendor-specific EDA flows
- universal hardware knowledge unrelated to SpinalHDL

## Evidence Sources

| Source | URL |
| --- | --- |
| SpinalHDL documentation (master) | https://spinalhdl.github.io/SpinalDoc-RTD/master/index.html |
| SpinalHDL source | https://github.com/SpinalHDL/SpinalHDL |
| SpinalHDL documentation source | https://github.com/SpinalHDL/SpinalDoc-RTD |
| VexRiscv | https://github.com/SpinalHDL/VexRiscv |
| NaxRiscv (newer plugin/fiber style) | https://github.com/SpinalHDL/NaxRiscv |

## Contributing

Contributions are welcome. If something is unclear, incomplete, or
wrong, please open an issue or a pull request. New evaluation prompts
that exercise real-world failures are especially valuable.

## License

MIT. See `LICENSE`.
