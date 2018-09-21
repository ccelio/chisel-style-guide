# chisel-style-guide

A Style Guide for the [Chisel Hardware Construction Language](https://chisel.eecs.berkeley.edu).


## Overall goal

Code is meant to be read, not written. You will spend more time searching for bugs, adding features to existing code bases, and trying to learn what other people have done, than you will writing your own code from scratch.  Code should strive to be easy to understand and easy to maintain.

As style can be a deeply personal preference, and because Chisel is still a very young language, this guide will eschew making hard edicts on DOs and DONTs. Instead, this guide will strive to provide guidance to newcomers to Chisel through a discussion on best practices.

## Prelude

Chisel is a DSL embedded in Scala. However, it is still a distinct language, and so it may not follow all of Scala's conventions.

For examples of Chisel style, I recommend reading the source code to the [Hwacha vector unit](https://github.com/ucb-bar/hwacha), the rocket-chip [uncore](https://github.com/ucb-bar/uncore), and the [BOOM out-of-order processor](https://github.com/ucb-bar/riscv-boom). Although they are all Berkeley-originated projects, each explores some very interesting (and different!) techniques to describing hardware generators.

Feedback requested!

## Table of Contents

* [Spacing](#spacing)
* [Naming](#naming)
* [Registers](#registers)
* [Bundles](#bundles)
* [Literals](#literals)
* [Parameters](#parameters)
* [Ready/Valid Interfaces](#ready-valid-interfaces)
* [Vector of Modules](#vector-of-modules)
* [Val versus Var](#val-versus-var)
* [Private versus Public](#private-versus-public)
* [Imports](#imports)
* [Comments](#comments)
* [Assertions](#assertions)
* [Requires](#requires)
* [Additional Best Practices](#additional-best-practices)

## Spacing

Spaces, not tabs. Never tabs.

Follow the indent level of the existing code.


## Naming

Variable names should tend to be descriptive and not overly abbreviated. The smaller the scope (and the more used it is), the more abbreviated the name can be.

Bundles used for I/Os should be named SomethingIO. IO comes last. The name is Camel-cased. Example: (FreeListIO).

Any variable that is used for debugging purposes should be begin with the prefix `debug_` (i.e., things that you ideally would not synthesize).

Constants/parameters should be named in all caps, demonstrating their global nature. While most things in Scala are immutable, Chisel isn’t Scala.

Constants should be all uppercase and should be put in a companion object:

````scala
    object ALU {
      val SZ_ALU_FN = 4
      FN_ADD = UInt(0, SZ_ALU_FN)
      ...
    }
````

Or trait (if you want a Module or Bundle to extend the trait):

````scala
    trait RISCVConstants {
       val RD_MSB  = 11
       val RD_LSB  = 7
    }
````

## Registers

Registers (and their type) should be specified as follows:

````scala
Reg(UInt())               // good!
Reg(UInt(width=8.W))      // also good!

Reg(io.my_signal.clone()) // good!
````

This construct `Reg(x)` tells the Reg to be of type `x`. It does **NOT** tell Reg what initial value it should be, nor does it add a `Delay` to a signal!

````scala
Reg(UInt(0))      // bad!  Returns a Reg of type UInt and unknown width.
Reg(UInt(0,15))   // bad!  Returns a Reg of type UInt with width 15.

Reg(io.my_signal) // bad. This makes a Reg of type io.my_signal, but the intention is not clear!
                  // It can be easily misread as Reg(next=io.my_signal).
````

Registers should be initialized as follows:

````scala
RegInit(UInt(0,15))   // good

Reg(UInt(0,15))       // WRONG! This is exactly equivelant to Reg(UInt(width=15)),
                      // and does NOT provide an initial value of UInt(0,15) to the Reg.
````
Delaying a Node (i.e., piping it into a register) should be performed as follows:

````scala
RegNext(io.my_signal)  // good
Reg(next=io.my_signal) // okay

Reg(io.my_signal)      // WRONG! Creates a Reg of the same type as io.a,
                       // and does NOT delay the node io.a with a register.
````

## Bundles

Consider providing `def` functions in your Bundles. It provides a clearer level of intention to the user of how to interact with the Bundle.

````scala
// simplified example
class DecoupledIO extends Bundle {
  val ready = Input(Bool())
  val valid = Output(Bool())
  def fire(dummy: Int = 0): Bool = ready && valid
  ....
````
Users of the DecoupledIO can now do something like `when(io.deq.fire())`!  (**note:** the `dummy: Int = 0` argument must be provided to functions with no arguments placed within Bundles, as Chisel is (currently) unable to differentiate between fields that are wires and fields that are functions with no arguments).

Or this example, which performs a `query` against a TLB address translation structure:

````scala
class TLBIO extends VMUBundle
{
  val req = Decoupled(new rocket.TLBReq)
  val resp = new rocket.TLBRespNoHitIndex().flip

  def query(vpn: UInt, store: Bool): Bool = {
    this.req.bits.vpn := vpn
    this.req.bits.asid := 0.U
    this.req.bits.passthrough := false.B
    this.req.bits.instruction := false.B
    this.req.bits.store := store

    this.req.ready && !this.resp.miss
  }
````

The particular example is quite interesting - the `query` function provides a clearer interface to the user, it automatically sets up the request signals, *and* it provides a combinational return value to the caller!

### Conditional I/O Fields

Consider breaking off Conditional I/O fields into a separate Bundles (FreeListRollbackIO and FreeListSingleCycleIO).

## Literals

Be careful of using Scala Ints to describe Chisel literals. `0xffffffff` is a 32-bit signed integer with value -1, and thus will throw an error when used as `UInt(0xffffffff, 32)`. Instead, use Strings to describe large literals:

````scala
    UInt("hffffffff", 32)
````

## Parameters

When instantiating an object from another package, explicitly name the arguments:

````scala
val s2d = Module(new hardfloat.RecFNToRecFN(inExpWidth = 8, inSigWidth = 24, outExpWidth = 11, outSigWidth = 53))
````

This safe-guards against the order (or the name) of parameters changing in an external package without your knowledge.

## Ready-Valid Interfaces

A ready signal denotes a resource is available/is ready to be utilized.

A valid signal denotes something is valid and *can* commit a state update (it *will* commit a state update if the corresponding ready signal is high).

**Performance tip:** a valid signal may often be a late arriving signal. Try to avoid using valid signals to drive datapath logic, and instead use valid signals to gate off state updates.

A valid signal **should not** depend on the ready signal (unless you really know what you are doing). This hurts the critical path and can create combinational loops if both sides get coupled.

## Vector of Modules

### Static Indexing

An array of modules can be instantiated as follows:

````scala
    val my_args = Seq(1,2,3,4)
    val exe_units = for (i <- 0 until num_units) yield {
       val exe_unit = Module(new AluExeUnit(args = my_args(i)))
       // any wiring or other logic can go here
       exe_unit
    }
````

You can provide different input parameters to each constructor as required (the above toy example shows different elements of `my_args` being provided to each `AluExeUnit`).

The disadvantage is you cannot index the collection using Chisel nodes (aka, you can not dynamically index the collection during hardware execution).  If you must use a Scala collection (for the first advantage), you can still use dynamic indexing by grabbing a Vec of the IOs:

````scala
    val exe_units_io = Vec(exe_units.map(_.io))
````

### Vec

If you need to index the vector of Modules using a Chisel node, you can also use the following structure:

````scala
    val table = Vec.fill(num_elements) {Module(new TableElement()).io}

    val idx = Wire(UInt())
    table(idx).wen := true.B // indexed by a Chisel node!
````

Note that `table` is actually a `Vec` of `TableElement` `I/O` bundles.


## Val versus Var

Only use `val`, unless you are an experienced Chisel programmer. Even then, only use `var` in constrained situations (try to abstract it within a function). The use of the `var` can make it difficult to reason about your design.

For context, a bit more background is needed. A hardware design described in Chisel is quite literally a Scala program that, when executed, generates a hardware graph composed of Chisel Nodes that is then passed to a back-end which generates a cycle-exact replica in either C++ or Verilog (or whatever other formats supported by the backend).

Thus, `val` and `var` denote Scala variables (well more exactly, `val` is an immutable value and `var` is a mutable variable).

````scala
    val my_node = Wire(UInt())
````

This is a Scala value called `my_node`, which points to a Chisel Node in the hardware graph that is a `Wire` of type `UInt`. The `my_node` value can only ever point to this particular Chisel Node in the graph.

````scala
    var node_ptr = Wire(UInt())
````

Uh oh. The Scala variable `node_ptr` is pointing to a Chisel node in the graph, but it can later be changed to point to a new Chisel node!

````Scala
    var node_ptr = io.a
    node_ptr := true.B
    node_ptr = io.b
    node_ptr := false.B
````

In the above (scary!) code,
* `node_ptr` first points to `io.a`,
* then uses the Chisel assignment operator `:=` to set `io.a` to `true.B`
* `node_ptr` is then changed to point to `io.b`
* and finally, `io.b` is set to `false.B`!

We get the following Verilog output:

````Verilog
module Hello(
    output io_a,
    output io_b
);

  assign io_b = 1'h0;
  assign io_a = 1'h1;
endmodule
````

Using `var` can make it difficult to reason about the circuit. **And be CAREFUL when mixing `=` and `:=`**! The `=` is a Scala assignment, and sets a `var` variable to point to a new Node in the graph. Meanwhile, `:=` is a Chisel assignment and performs a new assignment *to* the Chisel Node. This distinction is important! For example, Chisel conditional `when` statements are for conditionally assigning values to Chisel Nodes - **the scala `=` operator is invisible to `when` statements!!**

````Scala
    var my_node = io.a
    my_node := true.B // this sets io.a to "true"
    my_node = true.B  // this sets the Scala variable my_node to point to a Chisel node that is a literal true
````

Consider the incorrect code below, which tries to mix `when`, `var`, and `=` to perform an OR reduction:

````Scala
   val my_bits = Wire(Bits(width=n))
   var temp = false.B
   for (i <- 0 until n) {
      when (my_bits(i)) {
         temp = true.B // wrong! always returns true.
         temp := true.B // compiler error!
      }
   }
````

For the first statement `temp = true.B`, the Scala variable `temp` points to the Chisel node `true.B`, ignoring the when() statement.

For the second statement `temp := true.B`, a Chisel compiler error is thrown because the code is trying to reassign the node `false.B` to be `true.B`, which is nonsensical.

**Conclusion: don't mix `when` and `var`'s!**

For completness sake, the proper code for an OR reduction would be `my_bits.orR` (and no need to use var or when!).

### Valid uses of Vars

There are a few valid uses of var. One would be to generate cascading logic. For example, this locking arbiter from ChiselUtil:

````Scala
var choose = (n-1).U
for (i <- n-2 to 0 by -1) {
  choose = Mux(io.in(i).valid, i.U, choose)
}
chosen := Mux(locked, lockIdx, choose)
````

After each iteration of the Scala `for` loop, `choose` is pointing to a new node in the cascading Mux tree.

Another use is forward declaring Modules that are conditionally instantiated later.

````scala
var fpu: FPUUnit = null
if (has_fpu) {
   fpu = Module(new FPUUnit())
   ...
````

## Imports

Try to avoid wildcard imports. They make code more obfuscated and fragile.

````scala
    import rocket.{UseFPU, XLen}
    import cde.{Parameters, Field}
````

AVOID using `import` statements for bringing in new Module and Bundle definitions. Instead, explicitly invoke the namespace when instantiating the Module or Bundle. It makes the origin of the object clear.

````scala
//bad
import rocket._
...
val tlb = Module(new TLB())

//good
val tlb = Module(new rocket.TLB())
````

## Private versus Public

By default, all `val`s and `def`s are public in Scala. Label all `def`s private if the scope is meant to stay internal to the current object. This makes intention clearer.

## Comments

Consider commenting the use of the I/O fields (especially if there are unintuitive timings!). Chisel I/Os aren’t functions - it isn’t obvious how to interface with a Module to other programmers.

````scala
class CpuReq extends Bundle {
    val addr = UInt(width = ...)
    val cmd  = UInt(width = ...)
    val data = UInt(width = ...) // is sent the cycle after the request is valid
````

In fact, you may prefer to codify timings in the names of the signals themselves:
````scala
val io = new Bundle {
    // send read addr on cycle 0, get data out on cycle 2.
    val s0_r_idx = Input(UInt(width = index_sz.W))
    val s2_r_out = Output(UInt(width = fetch_width.W))
````


If it required cleverness to write, you should probably describe **why** it does what it does. The reader is never as smart as the writer. Especially when it’s yourself.

## Assertions

If you solve a bug, strongly contemplate what `assert()` could have caught this bug and then add it.

If you are using a one-hot encoding, guard it with asserts! Especially calls to `OHToUInt`.

````scala
assert(PopCount(updates_oh) <= 1.U, "[MyModuleName] ...")
````

Note which Module the assert resides in when authoring the failure string.

## Requires

Scala provides a `require()` function that will throw a Scala run-time error when compiling your Chisel hardware if the condition is not met.

Use `require()` statements to guard against unsupported parameter values in your hardware generators.

Use `require()` statements to codify your assumptions in your code (e.g., `require(isPow2(num_entries))` for logic that only works when `num_entries` is a power of 2).



## Additional Best Practices

If you ever write `+N`, ask yourself if the number will ever be `NonPow2` (and then you should write a `require` statement if the logic depends on `Pow2` properties!). For example, wrap-around logic will be needed to guard incrementing pointers to queues with `NonPow2` number of elements. Just like in software, overflows and array bounds overruns are scary!

Avoid use of `var`. If you do use `var`, try to abstract it into a function/object. If you don’t understand why `var` and `when()` don’t mix, then for the love of god AVOID `var` (See Val versus Var section).

If you solve a bug, strongly contemplate what `assert()` could have caught this bug and then add it.

Consider restraining any undefined hardware behavior. It makes writing asserts easier.  Undefined behavior may provide for a more efficient circuit, but a circuit that works is even more efficient!
