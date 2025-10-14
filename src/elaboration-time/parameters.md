# Elaboration-Time Parameters

Elaboration-time parameters are variables that must be provided at elaboration-/compile-time.
They must be passed from the upper/outer instantiating unit (These can simply be passed through too.).
You should never edit these directly, unless you konw what you are doing.
In most cases, these are already set by the BOOM core configuration.

BOOM is highly configurable by design.
It has a number of objects/classes that control how wide/many/deep many BOOM units are.

## `BoomCoreParams`
The `BoomCoreParams` class configure core-wide parameters.
These include the number of bytes fetched, the width of the ROB, the number of entries in the load and store queues, etc.
Changing these parameters has massive implications for the generated hardware and its resulting performance!

* `coreWidth`: The number of micro-ops that may be decoded, have registers renamed, and committed simultaneously.
  This is what allows the core to be superscalar.
  This parameter is effectively what allows for parallelism in the ROB, which trickles out to the other modules that interact with the ROB.
  <div class="note">
  By default <code>coreWidth</code> is set to be the same as the <code>decodeWidth</code>!
  This means that the maximum number of micro-ops that can be committed at the same time is exactly the same as the number of micro-ops that can be produced each cycle!
  </div>
  <div class="note">
  BOOM requires that <code>coreWidth</code> be less-than-or-equal to the <code>fetchWidth</code>.
  </div>

* `fetchWidth`: The number of bytes to be fetched from memory in a single burst.
  In reality, instruction bytes are actually fetched from an internal fetc buffer.
  <div class="note">
  This value <strong>must</strong> be a multiple of RISC-V's uncompressed instruction width, i.e. multiples of <code>4</code>.
  The configurations Boom ships with is only tested with a <code>fetchWidth</code> of <code>4</code> and <code>8</code>.
  </div>

* `decodeWidth`: The number of micro-ops the decoder is allowed to output in a single cycle.
* `issueWidth`: The total number of issue units available.
  In the case of the small BOOM configuration, there are just 3 issue units, a memory unit, an integer unit, and the FP unit.
  This is **not** the size of the issue buffer on each execution unit (that is the `IssueParams.numEntries`).
* `numRobEntries`: The number of entries in the ROB.
* `numLdqEntries`: The number of outstanding loads that are allowed to be queued for handling.
* `numStqEntries`: The number of outstanding stores that are allowed to be queued for handling.
* `enableSFBOpt`: Enable "Short-Forward Branch" optimizations.
    > This optimization improves IPC by recoding difficult-to-predict branches into internal predicated micro-ops (Zhao, 2015).

  This means that instead of trying to predict the branch, instructions on both sides of the branch are fed through and turned into micro-ops, and each micro-op has the branch's condition attached as a predicate.
  The micro-ops are executed normally, while the branch's condition is evaluated (the control-flow changing branch has been removed).
  When the condition's result is finally known, the predicated micro-op's predicate condition is updated.
  If the predicate condition was incorrect, then the micro-op's speculative execution is cleaned up and rolled back, while the other micro-op sequence is allowed to commit to architectural state.
  (See [Wikipedia's Predication page](https://en.wikipedia.org/wiki/Predication_(computer_architecture)) for more information.)

## `IssueParams`
The `IssueParams` class is used to have execution units inform the scheduler and ROB the resources and availability of each execution unit.
This is a relatively small parameter class, so we list all fields below:

* `dispatchWidth`: The number of micro-ops which may be dispatched/inserted to the issue queue per clock cycle.
* `issueWidth`: The number of micro-ops which may be issued to the execution unit (and removed from the issue queue) per clock cycle.
* `numEntries`: The number of entries that can be stored in this issue queue (the depth of the queue).
  These are micro-ops which are sitting idle while waiting for the respective execution unit's resources to become available.
* `iqType`: The type of the issue queue (integer, memory, floating-point, etc.).
  This is a bit-mask enumerated value used to determine which issue queue (and therefore which execution unit) the micro-op should be sent to.

## `FtqParameters`
This configures the fetch target queue in the core's front-end.
The queue's logic is discussed in greater detail in [the FTQ modules's section](../modules/ftq.md).
The parameter class is small enough to list below:

* `nEntries`: The number of entries in the FTQ.

## `BoomTileParams`

## Misc.
Throughout BOOM, there is the notion of "wakeup ports".
These are the number of ports to another module that are used to record the results of a micro-op's execution completion(?).
For example, the ROB's wakeup ports are used to set the number of other modules which will writeback into the ROB.
