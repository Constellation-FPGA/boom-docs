# Execution

We discuss execution as a whole subsystem instead of as a single stage of a micro-op's execution because there is a significant amount of complexity involved in having multiple micro-ops execute simultaneously.
This significantly complicates reading, writing, and reasoning about the execution model and behavior of BOOM.

<div class="note">
In out-of-order cores, execution occurs <strong>after</strong> the micro-op is dispatched to the ROB and before its results are committed.
</div>

The execution of a uop is handled by one of the [`ExecutionUnits`](../modules/execution-units.md).

TODO: How does the `FPPipeline` fit into here?
