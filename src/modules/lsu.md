# LSU (Load-Store Unit)

The Load-Store Unit (LSU) has a similar structure to the [Re-Order Buffer (ROB)](./rob.md).
The LSU has two (2) separate queues, one for loads and one for stores, abbreviated to `Ldq` and `Stq` in both this documentation and in the Chisel source code.
Each queue is can perform its operations out-of-order and has a maximum size determined by either `numLdqEntries` or `numStqEntries`, which are set as [Elaboration-Time Parameters](../elaboration-time/parameters.md).
So, just like the ROB, the LSU's Stq/Ldq use a `head`/`tail` index to control the buffer's insertion, completion, and submission to the memory system.

The `head` index is the next operation to be committed.
The `tail` index is the **next** index to have an operation inserted.
Both indices wrap around upon reaching the maximum size of the buffer.

The store queue has two extra indices:
  * `stq_execute_head`: This points to the next store to execute.
  * `stq_commit_head`: This points to the next store to commit.

In the store queue's context:
  * **execute** means to submit the memory request to the memory subsystem.
    This could include accessing the TLB, the PTW, etc.
    Eventually, the memory request will reach main memory.
  * **commit** means that the memory operation is complete and the LSU can now "retire" the load/store instruction and pass the results back to the ROB.

If a load is marked as succeeded, that means the data was loaded from memory and that the LSU has sent the loaded data back into the core's physical registers.

## General Flow
In the sections below, we discuss the typical flow for a load/store.
The [Stores](#stores) section below discusses some of the additional complexity surrounding stores in BOOM's LSU.
BOOM is built on top of Rocket's Hella cache, which uses the same ports for loads and stores, so each operation looks nearly the same.
The difference comes in the command that is passed to the cache (load or store).
If the operation is a store, then the data to be stored to memory must be provided.
If it is a load, then a dummy/`DontCare` value can be passed on the data lines of the request.

Note that by this point, the core does not know and/or care about the contents being loaded/store nor what instructions generated the load/store request.
So, both integer and floating-point loads/stores **must** go through the LSU at this point.

The LSU takes in an address to load from/store to, the data to store, and the load's destination **physical** register.
All of this information is used to "route" the memory request to the proper hardware.

If a load is nacked, it means that the dcache `nack`ed the load for the next stage.

### Physical Memory Access
If the memory address is a physical address (or virtual memory is turned off), then the process to work with memory is more straightforward.
  1. The load/store is inserted into the respective queue, if there is space in the queue.
  2. If the Hella cache state controller is in the proper state, then the memory request is sent out to where?
     TODO: Determine the Hella cache's state (`hella_state`) machinery and what it is for.
     TODO: How are requests routed to other hardware downstream of the LSU?
  3. Eventually, when the data memory system acknowledges the request, the load/store queue will have the `executed` flag on the entry set to `true.B`.

For a physical load:
  1. Entry inserted into load queue when the load's data is valid
  2. At the same time, the kind of load is determined (incoming vs. wakeup)
     TODO: What is the difference between incoming and wakeup for loads/stores?
  3. Goes to TLB?
  4. Goes to dmem as a request, which is a `BoomDCacheReq`, which tracks the address and data, along with if this is a Hella cache request.
     The request is marked as `s0_executing_loads = true.B` at the same time.
  5. The next cycle, `s1_executing_loads` gets the value of `s0_executing_loads`.
     At the same time, `s1_set_execute` is created and aliased with `s1_executing_loads`.
     ```scala
     val s1_executing_loads = RegNext(s0_executing_loads)
     val s1_set_execute     = WireInit(s1_executing_loads)
     ```
  6. It is likely that the data cache will not have the data immediately available, so it will `nack`.
     This prevents the load queue entry from being marked as executed (`ldq(i).bits.executed`).
     When the load is *not* `nack`ed, then the queue entry is marked as executed and the LSU can move onto the next entry.
     So after firing the load to `dmem`, `s1_executing_loads(i)` becomes `true.B`, which then gets passed into `ldq(i).bits.executed`.
     That same clock cycle, the dcache can come back with a `nack`, to invalidate the load queue entry.
     If the request *was* `nack`ed, then the queue entry's `executed` flag is cleared.
     Since the `executed` flag for a queue entry is now `false.B`, its request is retried.
     Eventually, the dcache should **not** `nack` the request, becuase the data *is* in the cache.

The signals to look at for the example immediately above (`i` is the load's assigned queue index):
  * `ldq(i).valid`
  * `will_fire_load_incoming(i)_will_fire`
  * `will_fire_load_wakeup(i)_will_fire`
  * `fired_load_incoming_REG`
  * `io.dmem.req.bits(i).valid`
  * `io.dmem.req.bits(i).bits.addr`
  * `io.dmem.req.valid`
  * `dmem.req.fire(i)`
  * `dmem.req(i).valid`
  * `s1_executing_loads(i)`
  * `ldq(i).bits.executed`
  * `nacking_loads(i)`
  * `io.dmem.nack(i).valid`
  * `older_nacked_REG`

### Virtual Memory Access
If a virtual address is given to the LSU, then the flow is more complicated than the physical access counterpart.
The reasoning is that the virtual address needs to be translated to a physical address before the core can get the actual data the user requested.

Translation is done in two steps:
  1. The LSU goes to the TLB to try to lookup the virtual-to-physical mapping.
     If there is a hit here, the physical address is returned to the LSU and the LSU issues the physical memory request.
  2. If the TLB misses, then the TLB sends a request onto the Page Table Walker (PTW) to traverse the page tables to find the virtual address's corresponding physical address.

### Stores
Stores are unique in BOOM because sending the store request to the memory system effectively commits the instruction.
For this reson, stores are only fired when the uop reaches the ***COMMIT*** stage in BOOM's pipeline.
Further, there are 2 queues for stores:
  1. The Store Address Queue (SAQ)
  2. The Store Data Queue (SDQ)

## Exceptions
There are 2 kinds of exceptions that the LSU knows about:
  1. Exceptions in the surrounding core.
     These is the `exception` field marked as `Input` on the `LSUCoreIO`.
     This signals is intended to let the top-level core inform the LSU that an exception has been thrown/is being taken and that the LSU needs to clean up.
  2. Exceptions caused by the LSU's handling of the requested operation.
     These are named `lxcpt`, marked as `Output` on the `LSUCoreIO`, and are passed directly back to the ROB.
     These are the exceptions that would be thrown when the load/store were to be committed, such as a page fault.

We focus on the 2nd kind of exception here, because the first (exceptions from outside the LSU) are not terribly special in the LSU, they just stop the LSU and force a clean-up.

We also note that the LSU can raise a "mini-exception", which are BOOM-specific exceptions that are **not** visible to software.
Mini-exceptions exist in the exception space and are assigned to `cause` registers, but are a special kind of exception handled by the core itself and will never be sent up to software as a trap.
The main example is BOOM's `MINI_EXCEPTION_MEM_ORDERING` mini-exception.
The [trap handling subsystem has more discussion on mini-exceptions](../subsystems/trap-handling.md#mini-exceptions).

Like in the ROB, the LSU needs to keep track of the "current oldest exceptional uop" because there is only a single exception port, so only one exception can be raised per clock cycle.
Tracking this information is done with the `r_xcpt` and `r_xcpt_valid` registers.

There are 2 cases where the "new oldest exceptional uop" must be recalculated:
  1. The incoming store address finds a faulting load.
     By the setup of the LSU, because the store is just incoming to the LSU and a load is faulting, the store is, by definition, younger.
  2. The incoming load/store *address* is exceptional.
     This means the incoming operation must be older than whatever is currently being tracked as the oldest exceptional uop.
