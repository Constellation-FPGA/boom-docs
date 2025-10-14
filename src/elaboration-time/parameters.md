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
