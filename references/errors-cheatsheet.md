# Errors Cheat Sheet

One-screen lookups for the most common SpinalHDL elaboration errors. Each
entry shows the message shape, the typical root cause, and the smallest
fix. Anchors link to the full doc page.

URL base: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/

## WIDTH MISMATCH

```
WIDTH MISMATCH on (toplevel/b : UInt[4 bits]) := (toplevel/a : UInt[8 bits]) at ...
```

**Root cause.** Assignment between signals of different widths, or an
operator producing a wider result than the destination. SpinalHDL never
narrows silently.

**Smallest fix.**

```scala
b := a.resized              // infer target width from b
b := a.resize(4)            // explicit
b := a(3 downto 0)          // explicit slice
result := (a +^ b).resized  // widening add followed by explicit resize
```

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/width_mismatch.html

## LATCH DETECTED

```
LATCH DETECTED from the combinatorial signal (toplevel/a : UInt[8 bits]) ...
```

**Root cause.** A combinational signal is conditionally assigned but has
no default; one or more branches leave it undefined.

**Smallest fix.**

```scala
val a = UInt(8 bits)
a := 0            // default first
when(cond) {
  a := 42
}
```

Alternative when latch behavior is genuinely desired: declare the signal
as a register and assign conditionally.

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/latch_detected.html

## COMBINATORIAL LOOP

```
COMBINATORIAL LOOP :
  Partial chain :
    >>> (toplevel/a : UInt[8 bits]) at PlayDev.scala:831 >>>
    >>> (toplevel/d : UInt[8 bits]) at PlayDev.scala:834 >>>
    >>> (toplevel/b : UInt[8 bits]) at PlayDev.scala:832 >>>
    >>> (toplevel/a : UInt[8 bits]) at PlayDev.scala:831 >>>
```

**Root cause.** Pure combinational signals form a cycle. Always a design
bug unless intentional (very rare; needs `noCombLoopCheck`).

**Smallest fix.** Break the loop with a register:

```scala
val tmp = RegNext(b | d) init(0)
a := tmp
```

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/combinatorial_loop.html

## ASSIGNMENT OVERLAP

```
ASSIGNMENT OVERLAP completely the previous one of (toplevel/a : UInt[8 bits]) ...
```

**Root cause.** Two unconditional assignments to the same signal in the
same scope. The second one fully erases the first; SpinalHDL refuses to
emit such code by default.

**Smallest fix.**

```scala
a := 42
when(something) {
  a := 66                // conditional override is fine
}

// Or, if the override is genuinely unconditional and intentional:
a := 42
a.allowOverride            // flag the next assignment as allowed to overlap
a := 66
```

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/assignment_overlap.html

## NO DRIVER ON

```
NO DRIVER ON (toplevel/a : UInt[8 bits]) ...
```

**Root cause.** A combinational signal influences the design but is never
assigned. Common after deleting code that used to drive it.

**Smallest fix.** Drive it, or delete its declaration and any users.

```scala
val a = UInt(8 bits)
a := 0
result := a
```

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/no_driver_on.html

## UNASSIGNED REGISTER

```
UNASSIGNED REGISTER (toplevel/a : UInt[8 bits]) ...
```

**Root cause.** A `Reg(...)` is created and used, but no path ever writes
it. The register would have stayed at reset value or X forever.

**Smallest fix.** Assign it, or delete it and its readers.

```scala
val a = Reg(UInt(8 bits)) init(0)
a := io.in
```

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/unassigned_register.html

## PARTIALLY ASSIGNED REGISTER

```
PARTIALLY ASSIGNED REGISTER ...
```

**Root cause.** Within a conditional block, only some bits of a register
are assigned. The rest can stay at random / previous values.

**Smallest fix.**

```scala
val reg = Reg(UInt(8 bits)) init(0)
when(cond) {
  reg := 0
  reg(3 downto 0) := 0xF       // override low bits intentionally
}
```

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/partially_assigned_register.html

## CLOCK CROSSING VIOLATION

```
CLOCK CROSSING VIOLATION from (toplevel/regA : UInt[8 bits]) to (toplevel/regB : UInt[8 bits]).
```

**Root cause.** A register in one clock domain drives a register in a
different (non-synchronous) domain through pure combinational logic.

**Smallest fix.** Use a documented CDC primitive:

```scala
// 1-bit level signal
val safe = BufferCC(regA(0), init = False)

// 1-bit pulse
val pulse = PulseCCByToggle(pulseA, cdA, cdB)

// Wide bus
val cc = StreamCCByToggle(streamA, cdA, cdB)
```

If the crossing is genuinely safe but undetectable by the checker, use
`crossClockDomain` on the destination register's domain — sparingly.

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/clock_crossing_violation.html

## HIERARCHY VIOLATION

```
HIERARCHY VIOLATION : signal ... is accessed outside of the current component's scope
```

**Root cause.** A signal is read or assigned across a Component boundary
in a way that the rules forbid. Allowed: read directionless signals of
your own component, read/assign IO of self and IO of children. Forbidden:
read internal signals of a child, write inputs of a parent, etc.

**Smallest fix.** Route the signal through an IO port of the component
that owns it; or move the offending logic into that component.

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/hierarchy_violation.html

## IO BUNDLE ERROR

```
IO BUNDLE ERROR : A direction less ... signal was defined into toplevel component's io bundle
```

**Root cause.** A signal inside `val io = new Bundle { ... }` lacks an
`in`/`out`/`inout` direction.

**Smallest fix.**

```scala
val io = new Bundle {
  val a = in  UInt(8 bits)
  val b = out Bool()
}
```

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/iobundle.html

## REGISTER DEFINED AS COMPONENT INPUT

```
REGISTER DEFINED AS COMPONENT INPUT : (toplevel/io_a : in UInt[8 bits]) ...
```

**Root cause.** `in(Reg(...))` in an `io` bundle. SpinalHDL disallows this
to avoid surprises in driver code.

**Smallest fix.**

```scala
val io = new Bundle {
  val a = in UInt(8 bits)
}
val aReg = RegNext(io.a)
```

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/register_defined_as_component_input.html

## UNREACHABLE IS STATEMENT

```
UNREACHABLE IS STATEMENT in the switch statement at ...
```

**Root cause.** Duplicate `is(value) { ... }` arms, or arms unreachable
because earlier arms cover the value.

**Smallest fix.** Delete the duplicate, or merge the bodies.

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/unreachable_is_statement.html

## OUT OF RANGE CONSTANT

```
OUT OF RANGE CONSTANT. Operator UInt < UInt
- Left  operand : (toplevel/value : in UInt[2 bits])
- Right operand : (U"101010" 6 bits)
```

**Root cause.** Comparing a narrow value against a literal that does not
fit in its width — the comparison is statically known.

**Smallest fix.** Widen the value, narrow the literal, or whitelist when
intentional:

```scala
(value < 42).allowOutOfRangeLiterals
```

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/out_of_range_constant.html

## SPINAL CAN'T CLONE CLASS

```
*** Spinal can't clone class ...
*** In place to declare a "class Bundle(args){}", create a "case class Bundle(args){}"
```

**Root cause.** SpinalHDL tried to clone a `Bundle` (typically for
`Stream(...)`, `Flow(...)`, `Reg(...)`) and could not retrieve the
constructor arguments because the class is not a `case class`.

**Smallest fix.** Convert to `case class`, or override `clone`.

```scala
case class RGB(width: Int) extends Bundle {
  val r, g, b = UInt(width bits)
}
```

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/spinal_cant_clone.html

## NULLPOINTEREXCEPTION (during elaboration)

```
Exception in thread "main" java.lang.NullPointerException at ...
```

**Root cause.** Almost always a Scala-level access of a `val` before its
initialization, often due to forward references in a `Component` body.

**Smallest fix.** Declare the signal before reading it; if forward
references are necessary, lift the body into a lazy `val` or `Area`.

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/nullpointerexception.html

## SCOPE VIOLATION

```
SCOPE VIOLATION : ... is assigned outside its declaration scope at ...
```

**Root cause.** A signal declared inside a `when`/`switch`/`Area` is
assigned outside of it via a captured `var`. Usually only happens when
metaprogramming is being a bit too clever.

**Smallest fix.** Declare the signal at the outer scope; do the
conditional binding without leaking the reference.

URL: https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/scope_violation.html

## How To Use This File

1. Copy the message line from the SpinalHDL output.
2. Search for it in this file (most messages start with a recognizable
   `UPPER CASE PHRASE`).
3. Apply the smallest fix; check the URL only if the cause is unclear.
4. If the message is not listed, fall back to the full index:
   https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Design%20errors/index.html
