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

## Why is this done this way?
From our understanding, having all floating-point operations contained within the `FpPipeline` allows for easier customization, since all FP logic is fully contained within a single module.
This allows all the FP-related ports connecting everything to be found easily by the Chisel compiler and optimized away by later passes.

## Loading and Storing
Instead of the `FPUExeUnit` handling FP loads/stores itself, it delegates a majority of the work.

Floating-point loads only go to the FP pipeline to write the loaded value.
The rest of the instruction does not touch any floating-point state, so it never goes through the FPU.
The integer load/store portion of the core handles floating-point loads.
The loaded value in the memory-handling execution units is connected to the FP pipeline's register file through the pipeline's `ll_wports`.
This connection happens at the core's top-level, in `core.scala`.

Floating-point stores are a little special because they are decoded into two uops.
The first is an `F2I` operation, which converts BOOM's floating-point numbers to traditional IEEE 754 floating-point numbers.
This is required because BOOM internally uses Hardfloat's recoded representation, not the IEEE one.
Second, the converted IEEE floating-point value is written out to memory.
To do this, the FP pipeline passes the destination address to the integer address generation unit (the `ALUExeUnit` where `hasMem = true`), then adds the converted IEEE value to the LSU's store queue.
