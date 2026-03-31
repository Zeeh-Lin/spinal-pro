# spinal-pro

`spinal-pro` is an agent skill for practical SpinalHDL development.

It helps AI coding agents generate, review, explain, and validate SpinalHDL RTL with a correctness-first workflow. The skill uses official SpinalHDL documentation for language semantics and mature SpinalHDL projects such as VexRiscv for architecture and engineering patterns.

This repository is packaged for OpenSkills, but it can also be used as a plain `SKILL.md` repository.

## When To Use

Use `spinal-pro` when the task involves:

- writing or modifying SpinalHDL RTL
- reviewing SpinalHDL for semantic bugs and structural risks
- explaining SpinalHDL DSL behavior such as `:=`, `when`, `ClockDomain`, `Stream`, `Flow`, `Bundle`, `Vec`, and FSMs
- designing parameterized modules, interfaces, buses, peripherals, or larger CPU/SoC structures
- validating SpinalHDL work through local generation, simulation, or repo-native tests

This skill is for practical SpinalHDL work, not general Scala or vendor-specific EDA flows unless SpinalHDL is central to the task.

## Quick Start

Install from GitHub:

```bash
npx openskills install Zeeh-Lin/spinal-pro --universal
npx openskills sync
```

## Without OpenSkills

If your agent already knows how to load Anthropic-style skills, it can read:

- `SKILL.md`
- files under `references/`

No extra OpenAI-specific metadata is required.

## Repository Layout

```text
spinal-pro/
├── README.md
├── SKILL.md
└── references/
    ├── architecture-patterns.md
    ├── language-semantics.md
    ├── review-checklist.md
    ├── rtl-recipes.md
    ├── simulation-validation.md
    └── vexriscv-casebook.md
```

- `SKILL.md`: the main operating contract for the agent
- `references/language-semantics.md`: correctness-critical SpinalHDL semantics
- `references/review-checklist.md`: review and bug-hunting checklist
- `references/rtl-recipes.md`: common implementation patterns
- `references/architecture-patterns.md`: larger design and decomposition patterns
- `references/simulation-validation.md`: build, simulation, and validation guidance
- `references/vexriscv-casebook.md`: a map from design questions to useful VexRiscv examples

## Scope

This skill focuses on:

- SpinalHDL DSL semantics
- practical RTL implementation patterns
- review guidance
- architecture patterns from real SpinalHDL codebases

It is intentionally narrow:

- keep `SKILL.md` short
- move detailed knowledge into `references/`
- prefer evidence over guesswork
- prefer official SpinalHDL docs for semantics
- use VexRiscv as a pattern source, not as a language spec

## Contributing

Contributions are welcome.

If something is unclear, incomplete, or wrong, please open an issue or send a pull request.

## License

MIT. See `LICENSE`.
