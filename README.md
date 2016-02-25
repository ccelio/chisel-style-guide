# chisel-style-guide

A Style Guide for the [Chisel Hardware Construction Language](https://chisel.eecs.berkeley.edu).


## Overall goal

code is meant to be read, not written. You will spend more time searching for bugs, adding features to existing code bases, and trying to learn what other people have done, than you will writing your own code from scratch.  Code should strive to be easy to understand and easy to maintain.

## Prelude

Chisel is a DSL embedded in Scala. However, it is still a distinct language, and so it may not follow all of Scala's conventions.

## Table of Contents

* [Spacing] (#spacing)
* [Naming] (#naming)
* [Bundles] (#bundles)
* [Ready/Valid Interfaces] (#ready-valid-interfaces)
* [Comments] (#comments)
* [Assertions] (#assertions)
* [Best Practices] (#best-practices)
 
## Spacing

Spaces, not tabs. Never tabs.

Follow the indent level of the existing code.


## Bundles

Consider breaking off Conditional I/O fields into a separate Bundles  (FreeListRollbackIO and FreeListSingleCycleIO).

## Naming

Variable names should tend to be descriptive and not overly abbreviated. The smaller the scope (and the more used it is), the more abbreviated the name can be.

Bundles used for I/Os should be named SomethingIO. IO comes last. The name is Camel-cased. Example: (FreeListIO).

Any variable that is used for debugging purposes should be named “debug_*” (i.e., things that you ideally will not synthesize).

Constants/parameters should be named in all caps, demonstrating their global nature. While most things in Scala are immutable, Chisel isn’t Scala.

Constants and should be put in a companion object.

    object Configuration {
      val NUM_ROB_ENTRIES = 32
    }

##Ready-Valid Interfaces

A ready signal denotes a resource is available/is ready to be utilized.

A valid signal denotes something is valid and *can* commit a state update (it *will* commit a state update if the corresponding ready signal is high).

A valid signal may often be a late arriving signal. Try to avoid using valid signals to drive datapath logic, and instead use valid signals to gate off state updates.

A valid signal **should not** depend on the ready signal (unless you really know what you are doing). This hurts the critical path and can create combinational loops if both sides get coupled. 


##Comments

Consider commenting the use of the I/O fields (especially if there are unintuitive and/or multi-cycle timings!). Chisel I/Os aren’t functions - it isn’t obvious how to interface with a Module to other programmers.

If it required cleverness to write, you should probably describe **why** it does what it does. The reader is never as smart as the writer. Especially when it’s yourself.

##Assertions

If you solve a bug, strongly contemplate what assert() could have caught this bug and then add it.

If you are using a one-hot encoding, guard it with asserts! Especially calls to OHToUInt. 

    (assert(PopCount(updates_oh) <= UInt(1), "[MyModuleName] ...")


##Best Practices

If you ever write `+N`, ask yourself if the number will ever be `NonPow2`.

**Avoid use of `var`**, as chisel lacks the defenses to safeguard its correct usage (Chisel nodes pointed to by `var` are invisible to `when()` statements). If you do use `var`, try to abstract it into a function/object. If you don’t understand why `var` and `when()` don’t mix, then for the love of god AVOID `var`.

If you solve a bug, strongly contemplate what `assert()` could have caught this bug and then add it.

Considering restraining any undefined hardware behavior. It makes writing asserts easier.  Undefined behavior may provide for a more efficient circuit, but a circuit that works is even more efficient!
