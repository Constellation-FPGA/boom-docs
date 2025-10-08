# Execution Units

ROB breaks from Rocket-Core by having a different configuration system for execution units.
Instead of having a dedicated Chisel module for each type of execution unit, BOOM combines them a little bit.

The `ExecutionUnits` abstraction is **just that**, an abstraction.
The main purpose of this abstraction is to allow for BOOM designers to use Scala's higher-order functions to operate on "sequences of execution units" while generating the expected hardware.
This means that unlike the `map`s used in Chisel code, `map`ping over the `ExecutionUnits` will ***not*** expand/generate additional hardware!

In a default BOOM configuration, you end up with three (3) execution units inside of `ExecutionUnits`:
  1. An ALU Execution unit.
     This performs all of the integer arithmetic and logic operations.
     This includes:
       * Jumps and branches
       * CSR manipulation
       * RoCC interfacing
       * Integer multiplication
       * Integer division
       * Integer register file to FP register file movements (if an FPU is requested, of course).
  2. A memory execution unit.
     The only purpose of the memory execution unit is to interact with the LSU to perform loads/stores.
     It does not perform any arithmetic (beyond generating load/store addresses).
  3. A [floating-point execution unit](#the-floating-point-fpuexeunit-execution-unit).
     This contains several functional units:
     * Floating-point FMA, to provide multiplication, addition, and subtraction.
     * Floating-point division and square root, if these operations are requested.
       (You can request an FPU where `usingFDivSqrt = false`.)
     * FP register file to integer register file movements.
     * Floating-point load/store?

Boom defined several "helper methods" to select which `ExecutionUnits` you may want to work with in a given scenario, including:
  * Units with ALUs (`ExecutionUnits.alu_units`)
  * Units with memory connections (`ExecutionUnits.mem_units`)
  * Units that interact with CSRs (`ExecutionUnits.csr_unit`)
  * and so on.

## Functional Units
Each of BOOM's *execution units* is made up of one or more *functional units*.
These are what allows a single execution unit to perform multiple actions.

## The Integer `ALUExeUnit` Execution Unit

## The Floating-Point `FPUExeUnit` Execution Unit
The entire design hierarchy to get to floating-point math is:
  1. `ExecutionUnits` (`execution-units.scala`) creates an `FPUExeUnit` (`execution-unit.scala`).
  2. Each `FPUExeUnit` creates multiple FP-related `FunctionalUnit`s (`functional-unit.scala`).
     Each `FunctionalUnit` is one of the ones required by the `FPUExeUnit`.
     For example, the `FPUUnit` functional unit (`functional-unit.scala`) provides fused multiply add (FMA), to support addition, subtraction, and multiplication.
  3. The `FPUUnit` (`functional-unit.scala`) creates an `FPU` (`fpu.scala`).
     This is a wrapper module that wraps Rocket-Chip's FPU so that BOOM can work with it.
     This includes a uop decoder too.
  4. BOOM's `FPU` (`fpu.scala`) create a module from Rocket-Chip's `tile` library, such as an `FPUFMAPipe`.

### Floating-Point (FPU)
The `ExecutionUnits` module creates an `FPUExeUnit` when the BOOM configuration requests one by setting `usingFPU = true`.
This provides two main components:
  * An FMA unit
  * A division/square root unit

The `FPUExeUnit` does ***NOT*** contain the floating-point register file!
The FPU execution unit is intended to **only** handle floating-point math, and nothing else!
To access the floating-point register file in the [FP Pipeline](./fp-pipeline.md), several read and write ports are created to access the actual register file.

#### Loading and Storing
RISC-V defines custom load/store instructions for floating-point values that write to/read from the floating-point register file directly.
The `FPUExeUnit` does **not** handle these, the [FP Pipeline](./fp-pipeline.md#loading-and-storing) does.
