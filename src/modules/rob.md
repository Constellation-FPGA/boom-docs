# ROB (Re-Order Buffer)

The Re-Order Buffer (ROB) is what allows BOOM's functional units to execute out-of-order while maintaining the illusion of sequential execution.
To achieve this, the ROB is a large buffer of micro-ops that are scheduled to execute by the scheduler.

TODO: Discuss the scheduler in the ROB!

The ROB uses three (3) indices to track entries in the ROB's buffer.
All of these indices wrap.
  * `head`: The oldest micro-op is at the head of the buffer and will be the next one committed.
  * `tail`: New micro-ops are inserted at the tail index.
  * `pnr`: "Point of No Return"

BOOM uses the "Point of No Return" (PNR) as a second commit head that runs ahead (has higher indices) of the actual commit head.
The PNR is responsible for marking the next instruction that might misspeculate or generate an exception.
These include unresolved branches and untranslated memory operations.
