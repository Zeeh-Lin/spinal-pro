# VexRiscv Casebook

Turn an abstract SpinalHDL design question into concrete VexRiscv
references. Each entry follows the same three-part shape:

- **Pattern** — what VexRiscv does, in one or two sentences
- **Preconditions** — when this pattern is safe to import
- **Anti-pattern** — what goes wrong when you copy it blindly

VexRiscv URL base: https://github.com/SpinalHDL/VexRiscv

Caveat: VexRiscv predates the official `spinal.lib.misc.pipeline` and
`spinal.lib.misc.plugin` libraries and uses its own framework. For new
SpinalHDL projects, prefer the official libraries (see
`architecture-patterns.md`) and use VexRiscv as a reasoning aid, not a
direct template.

## Core Map

| Topic | File | URL |
| --- | --- | --- |
| Pipeline skeleton + default `Stageable` set | `VexRiscv/src/main/scala/vexriscv/VexRiscv.scala` | https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/VexRiscv.scala |
| Plugin host, stage interconnect, inserted data flow | `VexRiscv/src/main/scala/vexriscv/Pipeline.scala` | https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/Pipeline.scala |
| Service contracts | `VexRiscv/src/main/scala/vexriscv/Services.scala` | https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/Services.scala |
| Plugin base type | `VexRiscv/src/main/scala/vexriscv/plugin/Plugin.scala` | https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/plugin/Plugin.scala |

## Configuration Examples

| Use case | File | URL |
| --- | --- | --- |
| Small, productive baseline | `demo/GenSmallAndProductive.scala` | https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/demo/GenSmallAndProductive.scala |
| Cached + MMU full config | `demo/GenFull.scala` | https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/demo/GenFull.scala |
| SoC composition, reset, APB peripherals | `demo/Murax.scala` | https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/demo/Murax.scala |

## Pattern Entries

### 1. Plugin-host CPU with two-phase elaboration

**Pattern.** A nearly empty top-level `VexRiscv` component owns a list of
plugins. Each plugin gets a `setup()` callback (request services, lock
peers) followed by a `build()` callback (materialize hardware) once all
setups have completed.

```scala
class MyPlugin extends Plugin[VexRiscv] {
  override def setup(pipeline: VexRiscv): Unit = {
    val decoder = pipeline.service(classOf[DecoderService])
    // request decoded fields, register defaults
  }
  override def build(pipeline: VexRiscv): Unit = {
    // emit hardware here
  }
}
```

**Preconditions.** Adopt this style when (a) feature swap-in / swap-out
matters more than top-level clarity, (b) you accept two-phase elaboration
discipline, and (c) you can maintain the service contracts.

**Anti-pattern.** Copying one plugin (say, `MulDivIterativePlugin`) into a
new design without bringing its `DecoderService`, `RegFileService`,
`HazardService` peers. Compilation may succeed; runtime will misroute
writes or mis-bypass values.

Prefer migration to `spinal.lib.misc.plugin.FiberPlugin` for new designs;
the model is more flexible (uses fibers instead of strict setup/build).

### 2. Stage-coupled execute / write-back plugins

**Pattern.** Functional plugins declare `Stageable[Bool]` flags
(`IS_ADD`, `IS_BRANCH`, …) on the decode stage and consume them later;
the plugin host pipelines them automatically between stages.

```scala
val IS_MY_OP = Stageable(Bool())
decoder.add(MY_OP, List(IS_MY_OP -> True, ...))
// later in build:
execute plug new Area {
  when(execute.input(IS_MY_OP)) { /* compute */ }
}
```

**Preconditions.** Use only if your target has VexRiscv-style stages
(`fetch`, `decode`, `execute`, `memory`, `writeBack`). Match where the
result is produced *and* where it is advertised as bypassable.

**Anti-pattern.** Producing a result in `writeBack` but registering it
with the hazard service as `execute`-stage bypassable. Manifests as
silent functional errors on dependent instructions.

Reference: `plugin/HazardSimplePlugin.scala` and
`plugin/Fetcher.scala` ·
https://github.com/SpinalHDL/VexRiscv/tree/master/src/main/scala/vexriscv/plugin

### 3. Custom instruction via decoder service

**Pattern.** Add a new SIMD or accelerator instruction by registering an
encoding with the decoder and adding a plugin that produces the result.

```scala
// In setup:
val decoder = pipeline.service(classOf[DecoderService])
decoder.add(
  key = MaskedLiteral("0000000_-----_-----_000_-----_0001011"),
  List(
    IS_MY_OP -> True,
    REGFILE_WRITE_VALID -> True,
    BYPASSABLE_EXECUTE_STAGE -> True
  )
)
```

**Preconditions.** You can express the encoding as a `MaskedLiteral`;
your operation either fits the execute stage timing budget or you accept
multi-cycle stalls; you have updated the register-file / bypass flags.

**Anti-pattern.** Flagging `BYPASSABLE_EXECUTE_STAGE -> True` while
producing the value in a later stage. Same failure mode as #2.

Reference: `demo/GenCustomSimdAdd.scala` ·
https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/demo/GenCustomSimdAdd.scala

### 4. Custom CSR

**Pattern.** Use `CustomCsrDemoPlugin` style: hook into `CsrService` to
expose a read/write CSR backed by your plugin's storage.

**Preconditions.** The target VexRiscv config includes `CsrPlugin`;
chosen CSR number does not collide with allocated RISC-V addresses or
other custom CSRs.

**Anti-pattern.** Allocating CSR numbers without checking the privileged
spec; using CSRs as a back door for combinational paths that should have
been MMIO.

Reference: `demo/CustomCsrDemoPlugin.scala` ·
https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/demo/CustomCsrDemoPlugin.scala

### 5. Frontend with prediction

**Pattern.** `IBusSimplePlugin` / `IBusCachedPlugin` define memory
shape; `BranchPlugin` defines where branches resolve and the penalty.
The frontend reads decoded data via the `Fetcher` service.

**Preconditions.** Any change to prediction logic must reconsider the
fetch buffer, redirect timing, and the documented
`Missing inserts : INSTRUCTION_ANTICIPATE` failure mode (see
`VexRiscv/README.md`).

**Anti-pattern.** Adjusting branch resolution stage without re-validating
the frontend's redirect latency; cargo-culting `IBusCachedPlugin` for a
target with no cache.

Reference: `plugin/Fetcher.scala`, `plugin/BranchPlugin.scala`,
`plugin/IBusSimplePlugin.scala`, `plugin/IBusCachedPlugin.scala`.

### 6. Debug and multi-domain reset

**Pattern.** `DebugPlugin` adds a debug clock domain with its own reset;
the SoC composes the core reset from system reset and debug reset.
`Murax.scala` shows the wiring.

**Preconditions.** Your SoC already has a discipline for splitting
debug-from-core resets; you can route the debug bus through to a probe.

**Anti-pattern.** Adding debug features that re-use the core reset, or
removing the debug domain "to simplify" without checking whether other
plugins consume `pipeline.debugReset` or its services.

Reference: `plugin/DebugPlugin.scala` ·
https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/plugin/DebugPlugin.scala

## Non-CPU Reference: GCD Peripheral

Use this cluster when the question is "how do I structure a small
control-plus-datapath peripheral with an APB interface and a focused sim"?
It is more readable than dropping into the full CPU.

| File | URL |
| --- | --- |
| Tutorial narrative | https://github.com/SpinalHDL/VexRiscv/blob/master/doc/gcdPeripheral/README.md |
| Shared `Bundle` and top wiring | https://github.com/SpinalHDL/VexRiscv/blob/master/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDTop.scala |
| Datapath | https://github.com/SpinalHDL/VexRiscv/blob/master/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDData.scala |
| FSM control path | https://github.com/SpinalHDL/VexRiscv/blob/master/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDCtrl.scala |
| APB register bank | https://github.com/SpinalHDL/VexRiscv/blob/master/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/Apb3GCDCtrl.scala |
| Focused simulation | https://github.com/SpinalHDL/VexRiscv/blob/master/doc/gcdPeripheral/src/main/scala/vexriscv/periph/gcd/GCDTopSim.scala |

## Do Not Cargo-Cult Checklist

Before reusing any VexRiscv pattern, verify:

- the destination has the same stage and timing assumptions
- the same reset and clock-domain composition applies
- the same service dependencies exist (or can be stubbed)
- the bus contract matches the destination environment
- the config knobs are still meaningful outside VexRiscv
- you have a smaller official-library alternative in mind for new code
  (`spinal.lib.misc.pipeline`, `spinal.lib.misc.plugin`,
  `spinal.lib.bus.*`)

Treat VexRiscv as a mature pattern library, not as a substitute for the
official SpinalHDL semantics. Language rules always come from
`language-semantics.md`.
