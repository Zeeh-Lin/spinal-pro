# Language Semantics

Correctness-critical SpinalHDL behavior. Every rule below is grounded in
the official documentation. When the bundled SpinalDoc-RTD checkout is not
available, follow the URL listed next to each anchor.

Target version: SpinalHDL 1.10+ (rules below were validated against the
`master` branch of SpinalDoc-RTD at the time of writing).

## Doc Anchors

| Topic | Local path | URL |
| --- | --- | --- |
| Assignments and width rules | `SpinalDoc-RTD/source/SpinalHDL/Semantic/assignments.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Semantic/assignments.html |
| Rules / last-assignment-wins | `SpinalDoc-RTD/source/SpinalHDL/Semantic/rules.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Semantic/rules.html |
| `when` / `switch` / `Mux` | `SpinalDoc-RTD/source/SpinalHDL/Semantic/when_switch.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Semantic/when_switch.html |
| Registers and reset | `SpinalDoc-RTD/source/SpinalHDL/Sequential logic/registers.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Sequential%20logic/registers.html |
| RAM/ROM `Mem` | `SpinalDoc-RTD/source/SpinalHDL/Sequential logic/memory.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Sequential%20logic/memory.html |
| Clock domains | `SpinalDoc-RTD/source/SpinalHDL/Structuring/clock_domain.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Structuring/clock_domain.html |
| `Bundle` / `IMasterSlave` | `SpinalDoc-RTD/source/SpinalHDL/Data types/bundle.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Data%20types/bundle.html |
| `Vec` | `SpinalDoc-RTD/source/SpinalHDL/Data types/Vec.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Data%20types/Vec.html |
| `Stream` | `SpinalDoc-RTD/source/SpinalHDL/Libraries/stream.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html |
| `Flow` | `SpinalDoc-RTD/source/SpinalHDL/Libraries/flow.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/flow.html |
| `StateMachine` | `SpinalDoc-RTD/source/SpinalHDL/Libraries/fsm.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/fsm.html |
| Design errors | `SpinalDoc-RTD/source/SpinalHDL/Design errors/*.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/index.html |

If both the local path and the URL are unreachable, say so in the response;
do not paraphrase from memory.

## Operator Cheat Sheet

| Operator | Meaning | Use on |
| --- | --- | --- |
| `:=` | Standard assignment, last valid assignment wins under `when`/`switch`. | combinational signals, registers |
| `\=` | Immediate in-place update, equivalent to Verilog blocking `=`. Reads after this line see the new value. | combinational signals only — does **not** work on `Reg(...)` |
| `<>` | Direction-driven structural connect. Resolves `in`/`out` from each side. | `Bundle` / `IMasterSlave` pairs and child component IO |
| `<<` / `>>` | Stream/Flow connect, no resize, no pipe. | `Stream[T]`, `Flow[T]` |
| `<-<` / `>-<` / `</<` / `<-/<` | Stream connect through `m2sPipe`/`s2mPipe`/both. | `Stream[T]` |
| `===` / `=/=` | Equality / inequality on any `Data`. | values and Bundles |

Signal nature is fixed at declaration. `UInt(8 bits)`, `Bool()`, `Bundle`, `Vec`
are combinational. Wrap with `Reg(...)`, `RegInit(...)`, `RegNext(...)`, or
`RegNextWhen(...)` to get a register. The way you later assign does not change
this.

## Normative Rules With Code

### 1. Combinational vs sequential is declaration-time

```scala
val a = UInt(4 bits)              // combinational
val b = Reg(UInt(4 bits))         // register, no reset
val c = Reg(UInt(4 bits)) init(0) // register, reset to 0
val d = RegNext(a)                // register, samples `a` each cycle
val e = RegNextWhen(a, enable)    // register, samples `a` when enable
```

### 2. Last-assignment-wins inside the same scope

```scala
val a = UInt(4 bits)
a := 0
when(cond) {
  a := 1   // overrides the 0 only while cond is true
}
// at this point: a == 1 if cond else 0

// Bad — both assignments are in the same scope, second one wipes the first.
val x = UInt(4 bits)
x := 0
x := 1   // ASSIGNMENT OVERLAP error
```

### 3. `\=` is Scala-level rebinding for read-after-write

```scala
var x = UInt(4 bits)
val y, z = UInt(4 bits)
x := 0
y := x      // y sees 0
x \= x + 1  // rebinds the Scala name x to a new combinational expression
z := x      // z sees 1
```

Only use `\=` on combinational signals. On a `Reg`, the underlying hardware
keeps existing while the Scala name now points elsewhere — almost always a bug.

### 4. Widths must match unless explicitly resized

```scala
val a = UInt(8 bits)
val b = UInt(4 bits)
b := a            // WIDTH MISMATCH
b := a.resized    // ok, inferred from b
b := a(3 downto 0) // ok, explicit slice
b := a.resize(4)   // ok, explicit width
```

The one automatic exception: weak-width literals widen automatically.

```scala
val out8 = UInt(8 bits)
out8 := U(3) // ok: U(3) is "weak width", widened to 8 bits
out8 := U(0x100) // error: literal needs 9 bits, narrowing not allowed
```

### 5. Give combinational signals a default

```scala
// Bad — LATCH DETECTED on a
val a = UInt(8 bits)
when(cond) { a := 42 }

// Good — default first, override conditionally
val a = UInt(8 bits)
a := 0
when(cond) { a := 42 }
```

### 6. Registers retain their value if not assigned

```scala
val acc = Reg(UInt(8 bits)) init(0)
when(io.push.fire) {
  acc := acc + io.push.payload
}
// no `else` needed: acc holds its previous value
```

If a register has consumers but never any assignment, SpinalHDL emits
`UNASSIGNED REGISTER`.

### 7. Registers capture their clock domain at creation, not at assignment

```scala
val coreCd = ClockDomain(io.clk, io.rst)
val coreArea = new ClockingArea(coreCd) {
  val r = Reg(UInt(8 bits)) init(0)  // belongs to coreCd
}
coreArea.r := io.something            // assignment outside does not change the domain
```

Use `setAsReg()` carefully: it creates the register in the clock domain of the
signal, not where `setAsReg()` is called.

### 8. `Stream` handshake contract

```scala
// Producer side
val out = master Stream (UInt(8 bits))
out.valid := True
out.payload := 0x42
// out.ready is driven by the consumer

// Consumer side
val in = slave Stream (UInt(8 bits))
in.ready := True
when(in.fire) {           // fire := valid && ready
  buffer := in.payload
}
```

Rules:
- A transfer happens only on a cycle where `valid && ready`.
- Once asserted, `valid` may only fall the cycle after a `fire`. The payload
  must not change while `valid` is high and the slave has not accepted it,
  except for documented exceptions (arbiters without lock, raw UART
  passthroughs).
- `valid` must not depend combinationally on `ready`. The reverse is fine.

### 9. `Flow` is `valid` + `payload` only — no backpressure

```scala
val ev = master Flow (UInt(8 bits))
ev.valid := pulse
ev.payload := count
```

Use `Flow` only when the consumer cannot stall. If you find yourself wanting a
`ready` line, switch to `Stream`.

### 10. `Bundle` + `IMasterSlave` defines a directional interface

```scala
case class Handshake(width: Int) extends Bundle with IMasterSlave {
  val valid   = Bool()
  val ready   = Bool()
  val payload = Bits(width bits)

  override def asMaster(): Unit = {
    out(valid, payload)
    in(ready)
  }
}

val io = new Bundle {
  val up   = slave  (Handshake(8))
  val down = master (Handshake(8))
}

io.down <> io.up  // direction-correct, signal-by-signal connect
```

If you forget to implement `asMaster()`, `<>` falls back to silent
direction-less behavior and you will get downstream wiring confusion.

### 11. `Vec` notation has two distinct shapes

```scala
// (a) Aggregate existing signals — creates no new hardware
val s0, s1 = UInt(4 bits)
val pair = Vec(s0, s1)         // pair(0) IS s0, pair(1) IS s1

// (b) Allocate a new vector of fresh signals
val regs = Vec(Reg(UInt(8 bits)), 4)
val wires = Vec.fill(8)(Bool())
```

Form (a) does not create hardware; assigning `pair(0)` mutates `s0`. Form (b)
allocates new signals or registers. Mixing them up is a common confusion.

### 12. `when` / `switch` defaults and exhaustiveness

```scala
val sel = UInt(2 bits)
val out = UInt(4 bits)

// Without a default the missing branch infers a latch
out := 0
switch(sel) {
  is(0) { out := 1 }
  is(1) { out := 2 }
  is(2) { out := 3 }
  is(3) { out := 4 }
}

// `default` covers everything else
switch(sel) {
  is(0)   { out := 1 }
  default { out := 0 }
}

// If your `is` arms already cover every value, dropping `default` is fine.
// Adding both triggers UNREACHABLE DEFAULT STATEMENT unless you opt in:
// switch(sel, coverUnreachable = true) { ... }
```

`if` / `else if` in elaboration time is fine; for runtime conditions use
`when` / `elsewhen` / `otherwise` to keep the priority encoding explicit.

### 13. `CombInit` makes an independent combinational copy

```scala
val a = UInt(8 bits)
a := 1

val b = a            // b is just another Scala name for a
val c = CombInit(a)  // c is a new wire that mirrors a, but is independent

when(sel) {
  b := 2  // also rewrites a — likely not what you wanted
  c := 2  // only c changes
}
```

Use `CombInit` when a helper function may want to override a returned value
without disturbing the caller's signal.

## High-Risk Semantic Checks

Apply extra scrutiny when the task touches:

- `ClockDomain`, `ClockingArea`, reset polarity, reset kind, soft reset, clock enable
- `Stream` / `Flow` bridges, FIFOs, custom `ready`/`valid` glue
- signed vs unsigned arithmetic, literal sizing, explicit `asUInt` / `asSInt` casts
- `setAsReg()`, especially when used across areas
- nested `when` / `switch` trees without obvious defaults
- `Mux` / `MuxList` logic that might not be exhaustive
- manual overrides like `allowOverride`, `allowUnsetRegToAvoidLatch`, comb-loop suppression

Trigger phrases that should put the agent into "explain assumptions
explicitly" mode: `.resized`, `.asUInt`, `.asSInt`, `setAsReg`, `allowOverride`,
`ClockDomain(...)`, `ClockingArea`, `BufferCC`, `crossClockDomain`,
`unsafeAssignFromBits`, raw `Mem(...).readWriteSync(...)` with
`readUnderWrite = writeFirst`, `randBoot`.

## Error-Driven Reading Guide

See `errors-cheatsheet.md` for one-screen lookups. The full doc anchors are:

| Error message | Local | URL |
| --- | --- | --- |
| `WIDTH MISMATCH` | `Design errors/width_mismatch.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/width_mismatch.html |
| `LATCH DETECTED` | `Design errors/latch_detected.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/latch_detected.html |
| `COMBINATORIAL LOOP` | `Design errors/combinatorial_loop.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/combinatorial_loop.html |
| `ASSIGNMENT OVERLAP` | `Design errors/assignment_overlap.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/assignment_overlap.html |
| `NO DRIVER ON` | `Design errors/no_driver_on.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/no_driver_on.html |
| `UNASSIGNED REGISTER` | `Design errors/unassigned_register.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/unassigned_register.html |
| `CLOCK CROSSING VIOLATION` | `Design errors/clock_crossing_violation.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/clock_crossing_violation.html |
| `HIERARCHY VIOLATION` | `Design errors/hierarchy_violation.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/hierarchy_violation.html |
| `IO BUNDLE ERROR` | `Design errors/iobundle.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/iobundle.html |
| `REGISTER DEFINED AS COMPONENT INPUT` | `Design errors/register_defined_as_component_input.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/register_defined_as_component_input.html |
| `PARTIALLY ASSIGNED REGISTER` | `Design errors/partially_assigned_register.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/partially_assigned_register.html |
| `UNREACHABLE IS STATEMENT` | `Design errors/unreachable_is_statement.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/unreachable_is_statement.html |
| `OUT OF RANGE CONSTANT` | `Design errors/out_of_range_constant.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/out_of_range_constant.html |
| `Spinal can't clone class` | `Design errors/spinal_cant_clone.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/spinal_cant_clone.html |
| `NullPointerException` (elab-time) | `Design errors/nullpointerexception.rst` | https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/nullpointerexception.html |

## Practical Interpretation

- Use official docs to answer *what does this construct mean in SpinalHDL*.
- Use the local repository to answer *how is this codebase already doing it*.
- Use VexRiscv and similar projects only as pattern evidence; do not promote
  their stylistic choices to language rules.
