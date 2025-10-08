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
