# FP Pipeline

BOOM's FP Pipeline (`FpPipeline`) is intended to orchestrate all aspects of floating-point, not perform FP math.
In many ways, the FP pipeline resembles an pipelined in-order processor, with multiple distinct stages that connect together.

<div class="warning">
The FP Pipeline does <strong><em>NOT</em></strong> perform floating-point math!!
To see the modules that wrap and implement the actual logic of IEEE floating-point math, see <a href="./execution-units.html#floating-point-fpu">Floating-Point (FPU)</a>.
</div>

The FP pipeline handles:
  * Connecting floating-point operations to other modules
    - Connecting FP operations to address generators for loads/stores.
    - Connecting FP operations to the [LSU](./lsu.md) for submitting load/store requests.
  * Issuing FP operations to other FP modules
  * The floating-point register file (`frf` or `fregfile`)
  * Controlling and performing FP arithmetic with the FP execution units

## Loading and Storing
Instead of the `FPUExeUnit` handling FP loads/stores itself, it delegates a majority of the work to the integer address generation unit (the `ALUExeUnit` where `hasMem = true`), then adds to the LSU's load/store queues.

This connection happens at the core's top-level, in `core.scala`.
The FP pipeline's `ll_wports` are connected to the memory-handling execution units.
