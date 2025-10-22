# Register Renamer

<!-- toc -->

The register renamer (`RenameStage`) is the hardware module that tracks the mapping of an instruction's architectural registers to the micro-op's physical registers.
Boom's register renamer is broken up into several different pieces, all pulled together by the [`RenameStage`](#renamestage) module.

The renamer has many inputs and relatively few outputs, all of which are defined in the [`AbstractRenameStage`](#abstractrenamestage) module.
The renamer attaches to (has inputs):
  * The decoder stage (`io.dec_*`)
  * The dispatch stage (`io.dis_*`)
  * The commit (and therefore rollback) stage (`io.com_*` and `io.rbk_*`/`io.rollback`, respectively)

All of these are inputs because each of them needs to either consult or modify the rename mapping in some way.

The only outputs (that are not debugging outputs) are the `io.ren2_*` IO buses.
These outputs are only connected to the decode input wires.

## `AbstractRenameStage`
The `AbstractRenameStage` pipelines the rename stage and connects some of the inputs to outputs accordingly.
However, this abstract renamer ***DOES NOT*** have any renaming mechanism attached to it!
This means you cannot create a hardware instance of the `AbstractRenameStage` (and Scala will prevent you because the class is marked as `abstract`).
You are expected to instantiate the [`RenameStage`](#renamestage) module to achieve the desired renaming behavior.

Any register renamer that extends and implements the abstract base class will be a two-cycle pipelined renamer.
The two cycles/stages are called rename-1 and rename-2 (`ren1` and `ren2`) through this documentation and the source code itself.

## `RenameStage`
The `RenameStage` is the default module used to implement register renaming in BOOM.
It ties together the three main structures: the [map table](#renamemaptable), the [free list](#renamefreelist), and [busy table](#renamebusytable) to rename architectural registers to physical registers correctly and safely.

The register renamer for the integer and floating-point register files has a busy table **and** a free list.
The reasoning is as follows:
  * A register is busy if it has not had its final result written back to it yet.
    This means that later uops which **read** this physical register ***cannot*** be issued to execution units yet.
  * A register is free if it has not been allocated to a uop as a destination yet.
  * A register is not busy and not free when the final result has been written back, but the register is not yet stale and ready to be freed.
  * A register can never be busy and free at the same time.

### Register Renaming Process

1. Clock cycle 1 (`ren1_*`)
   1. Decoded instruction's uop is submitted to renamer.
   2. Renamer looks at logical (ISA) registers **immediately**, looking them up in the map table, returning whatever values were already in the [map table](#renamemaptable).
   Importantly, the renamer sets the uop's `stale_pdst` to the map table's returned value.
   A previous instruction *must* have allocated a register for the current one to read it (if not done, this is undefined behavior), so there are no `stale_prsN` values on the uop.
2. Clock cycle 2 (`ren2_*`).
   This stage allocates a new physical register from the renamer for the logical destination (`ldst`) to use as the *actual* physical destination (`pdst`), replacing the old `stale_pdst` found earlier.
   This is a complicated clock cycle, but bear in mind that all steps below ***happen simultaneously***!
   1. "Pop" from the [free list](#renamefreelist).
      This is done in a single clock cycle, because free list module is always outputting the **next** free registers.
      The free list updates its internal register on the **next** clock cycle.
   2. Mark the entry "popped" from the free list as busy in the [busy table](#renamebusytable).
      Again, this is done in a single clock cycle, because the free list is always outputting the **next** free registers, which means the busy table has these register indices as inputs all the time.
      The busy table updates its internal register on the **next** clock cycle.
   3. ***RE-MAP*** the [map table](#renamemaptable)'s entry for the uop's `ldst` from what the map table returned in `ren1` to the value(s) just popped from the free list, setting the uop's `pdst`.

## `RenameFreeList`
The rename free list is responsible for tracking which physical registers are free.
It does this by using a bitmask and several one-hot encoders.

For the example below assume we are operating on an N-wide core and are requesting a free register.
Everything in this example generalizes to wider cores too.
  1. Submit the request to the `RenameFreeList` module.
  2. Select the first N bits that are `true.B`/`1.U` in the `free_list`.
  3. Zip that with the subset of N ways to produce a mask of selections that are allowed to return a free register.
  4. Use this mask to update the free list.

This process needs to be aware of speculatively executed branches, so each speculative branch slot is also updated to reflect all non-speculative register removals from the free list.

An minor detail is that the free list needs to know how many registers should be "logically visible", the `numLregs`.
Technically, this fact should be used to differentiate the floating-point free list from the integer, since the FP register file does not have a hard-wired zero register.
However, in the current implementation of BOOM, this is not the case.

In the
```scala
class RenameFreeList(
  val plWidth: Int,
  val numPregs: Int,
  val numLregs: Int)
  (implicit p: Parameters) extends BoomModule
{ ... }

  // The free list can track numPhysRegs physical registers, but only 31 (32 if
  // floating-point) are available to be tracked as free/not-free.
  val freelist = Module(new RenameFreeList(
    plWidth,
    numPhysRegs,
    if (float) 32 else 31))
```

### Returning Registers
When a uop is committed by the ROB, the free list **immediately** clears the allocation flag for the uop's `stale_pdst`, if it is **not busy**, marking the register as free again.

<div class="warning">
A physical register is returned after being assigned to <strong><em>2</em></strong> micro-ops!
</div>

For example, say the ROB is committing a uop that has the following destination physical registers assigned to it: `stale_pdst = 0x0E` and `pdst = 0x21`.
If the ROB is in its normal state, then the register renamer is supposed to free the `stale_pdst`, since no uops can possibly refer to it anymore.
This means that the renamer will mark `0x0E` as free and not busy.
However, if the ROB is rolling-back, then the `pdst` will be freed and marked as not busy, since that physical register's contents should be discarded during rollback.
The next micro-op that is assigned `stale_pdst = 0x21` will free `0x21` when **it** commits.

## `RenameBusyTable`
The busy table is responsible for tracking which physical registers are currently in-use (busy) by other micro-ops.
This is done with a simple bitmask whose width matches the number of physical registers in the physical register file.

When a uop is committed by the ROB, the busy table **immediately** clears the busy flag, marking the register as not-busy.
Note that this behavior ***IDENTICAL*** to the [free list's](#renamefreelist) behavior!

The busy table also tracks uop source registers that are bypassed.
In our current version of Boom, bypassing through the busy table is disabled by default.

Overall, the core logic in the busy table and free list are very similar, but the busy table only tracks registers that are currently in-use, which means that the design is simpler when speculation happens (since a uop that is speculated still needs to mark the register as busy).

The busy table also has the ability to detect if the input uop that is undergoing renaming is bypassed as well.

### Un-busy-ing Registers
When a uop is committed by the ROB, the busy table **immediately** clears the allocation flag for the uop's `pdst`, marking the register as not busy.


## `RenameMapTable`
Finally, the rename mapping table!
This mapping table does exactly what you think it does; it is responsible for tracking which allocated physical register an architectural register has been renamed to.
The table maps logical (architectural) registers to physical registers.

<div class="note">
The zeroth <strong>physical</strong> register is <strong>not</strong> used!
The map table needs a default value to indicate the lack of a mapping, and physical register index <code>0</code> is this indicator.
</div>

An important detail is that the map table needs to know how many registers should be "logically visible", the `numLregs`.
When the map table is being used to track the integer register file, `numLregs` ensures that the zeroth register in the physical table is always pointing to value zero.

```scala
class RenameMapTable(
  val plWidth: Int,
  val numLregs: Int,
  val numPregs: Int,
  val bypass: Boolean,
  val float: Boolean)
  (implicit p: Parameters) extends BoomModule
{ ... }

  // Map table has numPhysRegs physical registers available that must be mapped
  // to 32 logical registers.
  val maptable = Module(new RenameMapTable(
    plWidth,
    32,
    numPhysRegs,
    false,
    float))
```

## `PredRenameStage`
The `PredRenameStage` is a specialty "register renamer" module intended ***ONLY*** for the [predication register file (`pregfile`)](./regfile.md#predicate-register-file-pregfile) in the top-level core's front-end.
If the user specifies the `enableSFBOpt` this module will provide register renames, otherwise, the module mostly functions as a pass-through module.
Importantly, this module does not use a free list but it does use a busy table.
This is because a predicated micro-op's physical registers must always be considered free, but they may be busy from other earlier predicated micro-ops.

The register renamer for the predicate file is different than the register renamers for the other physical registers because the predicate renamer only uses a busy table.
There is no free list because this register renamer does not need to support out-of-order allocation of registers.
By the nature of branches, all branches must be seen by the front-end in-order, which translates to converting a branch into a set-flag micro-op in-order.
Because the front-end must end up seeing these uops in-order, the [Fetch Target Queue](./ftq.md#ftq-fetch-target-queue) index can be used as the physical register index for the set-flag uop.
The FTQ index is a saturating counter whose width is determined by the [`FtqParameters`](../elaboration-time/parameters.md#ftqparameters) object (16 entries by default in our Boom designs).

The busy table in this module fulfills the same purpose as the [busy table](#renamebusytable) for the other register files.
When the predicate physical register file has the predicate's result written back to it, the corresponding busy flag is cleared.
