# chisel-style-guide

A Style Guide for the [Chisel Hardware Construction Language](https://chisel.eecs.berkeley.edu).


## Overall goal

Code is meant to be read, not written. You will spend more time searching for bugs, adding features to existing code bases, and trying to learn what other people have done, than you will writing your own code from scratch.  Code should strive to be easy to understand and easy to maintain.

As style can be such a deeply personal preference, and as Chisel continues to evolve, this guide will eschew making hard edicts on DOs and DONTs. Instead, this guide will strive to provide guidance to newcomers to Chisel through a discussion on best practices.

## Prelude

Chisel is a DSL embedded in Scala. However, it is still a distinct language, and so it may not follow all of Scala's conventions.

## Table of Contents

* [Spacing] (#spacing)
* [Naming] (#naming)
* [Bundles] (#bundles)
* [Literals] (#literals)
* [Ready/Valid Interfaces] (#ready-valid-interfaces)
* [Vector of Modules] (#vector-of-modules)
* [Val versus Var] (#val-versus-var)
* [Private versus Public] (#private-versus-public)
* [Imports] (#imports)
* [Comments] (#comments)
* [Assertions] (#assertions)
* [Best Practices] (#additiona-best-practices)
 
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
    object ALU 
    {
      val SZ_ALU_FN = 4
      FN_ADD = UInt(0, SZ_ALU_FN)
      ...
    }
````

Or trait (if you want a Module or Bundle to extend the trait):

````scala
    trait RISCVConstants
    {
       val RD_MSB  = 11
       val RD_LSB  = 7
    }
````

## Bundles

Consider providing `def` functions in your Bundles. It provides a clearer level of intention to the user of how to interact with the Bundle.

````scala
// simplified example of the DecoupledIO
class DecoupledIO extends Bundle                 
{                                                                    
  val ready = Bool(INPUT)                                            
  val valid = Bool(OUTPUT)                                           
  def fire(dummy: Int = 0): Bool = ready && valid     
  ....
````


Or

````scala
class TLBIO extends VMUBundle 
{                           
  val req = Decoupled(new rocket.TLBReq)                  
  val resp = new rocket.TLBRespNoHitIndex().flip          
                                                          
  def query(vpn: UInt, store: Bool): Bool = 
  {             
    this.req.bits.vpn := vpn                              
    this.req.bits.asid := UInt(0)                         
    this.req.bits.passthrough := Bool(false)              
    this.req.bits.instruction := Bool(false)              
    this.req.bits.store := store                          
                                                          
    this.req.ready && !this.resp.miss                     
  }                                                       
````

Consider breaking off Conditional I/O fields into a separate Bundles  (FreeListRollbackIO and FreeListSingleCycleIO).

## Literals

Be careful of using Scala Ints to describe Chisel literals. `0xffffffff` is a 32-bit signed integer with value -1, and thus will throw an error when used as `UInt(0xffffffff, 32)`. Instead, use Strings to describe large literals:

    UInt("hffffffff", 32)

##Ready-Valid Interfaces

A ready signal denotes a resource is available/is ready to be utilized.

A valid signal denotes something is valid and *can* commit a state update (it *will* commit a state update if the corresponding ready signal is high).

**Performance tip:** a valid signal may often be a late arriving signal. Try to avoid using valid signals to drive datapath logic, and instead use valid signals to gate off state updates.

A valid signal **should not** depend on the ready signal (unless you really know what you are doing). This hurts the critical path and can create combinational loops if both sides get coupled. 

##Vector of Modules

###ArrayBuffer

An ArrayBuffer should be used to describe a vector of Modules.

````scala
    val exe_units = ArrayBuffer[ExecutionUnit]()
    val my_args = Seq(1,2,3,4)
    for (i <- 0 until num_units)
    {
       exe_units += Module(new AluExeUnit(my_args(i)))
    }
````

One of the advantages is that you can provide different input parameters to each constructor (the above toy example shows different elements of `my_args` being provided to each `AluExeUnit`).

The disadvantage is you cannot index the ArrayBuffer using Chisel nodes (aka, you can not dynamically index the ArrayBuffer during hardware execution).

###Vec

If you need to index the vector of Modules using a Chisel node use the following structure:

````scala
    val table = Vec.fill(num_elements) {Module(new TableElement()).io}
     
    val idx = Wire(UInt())
    table(idx).wen := Bool(true) // indexed by a Chisel node!
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
    node_ptr := Bool(true)
    node_ptr = io.b
    node_ptr := Bool(false)
````

In the above (scary!) code, 
* `node_ptr` first points to `io.a`, 
* then uses the Chisel assignment operator `:=` to set `io.a` to `Bool(true)`
* `node_ptr` is then changed to point to `io.b`
* and finally, `io.b` is set to `Bool(false)`!

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

Using `var` can make it difficult to reason about the circuit. **And be CAREFUL when mixing `=` and `:=`**! The `=` is a Scala assignment, and sets a `var` variable to point to a new Node in the graph. Meanwhile, `:=` is a Chisel assignment and performs a new assignment *to* the Chisel Node. This distrinction is important! For example, Chisel conditional `when` statements are for conditionally assigning values to Chisel Nodes - **the scala `=` operator is invisible to `when` statements!!**

````Scala
    var my_node = io.a
    my_node := Bool(true) // this sets io.a to "true"
    my_node = Bool(true)  // this sets the Scala variable my_node to point to a Chisel node that is a literal true
````

Consider the incorrect code below, which tries to mix `when`, `var`, and `=` to perform an OR reduction:

````Scala
   val my_bits = Wire(Bits(width=n))
   var temp = Bool(false)
   for (i <- 0 until n)
   {
      when (my_bits(i))
      {
         temp = Bool(true) // wrong! always returns true.
         temp := Bool(true) // compiler error!
      }
   }
````

For the first statement `temp = Bool(true)`, the Scala variable `temp` points to the Chisel node `Bool(true)`, ignoring the when() statement.

For the second statement `temp := Bool(true)`, a Chisel compiler error is thrown because, because the code is trying to reassign the node `Bool(false)` to be `Bool(true)`, which is nonsensical. 

**Conclusion: don't mix `when` and `var`'s!**

For completness sake, the proper code for an OR reduction would be `my_bits.orR` (and no need to use var or when!). 

###Valid uses of Vars

There are a few valid uses of var. One would be to generate cascading logic. For example, this locking arbiter from ChiselUtil:

````Scala
var choose = UInt(n-1)                         
for (i <- n-2 to 0 by -1) {                    
  choose = Mux(io.in(i).valid, UInt(i), choose)
}                                              
chosen := Mux(locked, lockIdx, choose)         
````

After each iteration of the Scala `for` loop, `choose` is pointing to a new node in the cascading Mux tree.

Another use is forward declaring Modules that are conditionally instantiated later.

````scala
var fpu: FPUUnit = null                   
if (has_fpu)                                              
{                                                         
   fpu = Module(new FPUUnit()) 
   ...
````

##Imports

Try to avoid wildcard imports. They make code more obfuscated and fragile.

    import rocket.{UseFPU, XLen}
    import cde.{Parameters, Field}

##Private versus Public

By default, all `val`s and `def`s are public in Scala. Label all `def`s private if the scope is meant to stay internal to the current object. This makes intention clearer.

##Comments

Consider commenting the use of the I/O fields (especially if there are unintuitive timings!). Chisel I/Os aren’t functions - it isn’t obvious how to interface with a Module to other programmers.

    class CpuReq extends Bundle
    {
        val addr = UInt(width = ...)
        val cmd  = Bits(width = ...)
        val data = Bits(width = ...) // is sent the cycle after the request is valid

If it required cleverness to write, you should probably describe **why** it does what it does. The reader is never as smart as the writer. Especially when it’s yourself.

##Assertions

If you solve a bug, strongly contemplate what `assert()` could have caught this bug and then add it.

If you are using a one-hot encoding, guard it with asserts! Especially calls to `OHToUInt`. 

    assert(PopCount(updates_oh) <= UInt(1), "[MyModuleName] ...")

Note which Module the assert resides in when authoring the failure string.

##Additional Best Practices

If you ever write `+N`, ask yourself if the number will ever be `NonPow2`. For example, wrap-around logic will be needed to guard incrementing pointers to queues with `NonPow` number of elements. Just like in software, overflows and array bounds overruns are scary!

**Avoid use of `var`**, as chisel lacks the defenses to safeguard its correct usage (Chisel nodes pointed to by `var` are invisible to `when()` statements). If you do use `var`, try to abstract it into a function/object. If you don’t understand why `var` and `when()` don’t mix, then for the love of god AVOID `var`.

If you solve a bug, strongly contemplate what `assert()` could have caught this bug and then add it.

Considering restraining any undefined hardware behavior. It makes writing asserts easier.  Undefined behavior may provide for a more efficient circuit, but a circuit that works is even more efficient!
