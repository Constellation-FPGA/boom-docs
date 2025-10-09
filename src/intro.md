# Introduction

(Sonic)BOOM is a multi-stage pipelined, out-of-order, non-superscalar (?), speculative CPU design that implements RV32/64 GCV, along with support for the standard 3-privilege level model (M-, S-, and U-modes).
As of 2025-03-05, the authors are not sure BOOM fully and correctly implements the ratified 4-privilege level model (M-, H-, VS-, VU-modes).

<div class="warning">
<strong>We are not the original authors of BOOM!</strong><br>
All of the documentation present in this Wiki is gathered through testing, trial-and-error, manually reading through the code, and the authors' prior knowledge of computer architecture.
</div>

You may notice that we have split the discussion of subsystems and pipeline stages up.
This is because the two ideas are somewhat orthogonal in BOOM, because of its out-of-order nature.

If we ignore the always-necessary fetch, decode, rename, issue, and register read stages, then even a simple integer ALU instruction requires 3 distinct pipeline stages to complete.
  1. [Execute](./stages/exe.md): Compute the result.
  2. [Writeback](./stages/wb.md): Store result back to the physical register file.
  3. [Commit](./stages/com.md): The physical register with the result must be committed to the architecture register so that software can see the loaded value.

Load instructions requires 4 pipeline stages to complete:
  1. [Execute](./stages/exe.md): To calculate the effective load address.
  2. [Memory](./stages/mem.md): To submit the load request to the memory & caching subsystem.
  3. [Writeback](./stages/wb.md): The value loaded from memory must be written back to the physical register file.
  4. [Commit](./stages/com.md): The loaded physical register must be committed to the architecture register so that software can see the loaded value.

Further, some pipeline stages are discussed in a single section, despite them being split up into multiple clock cycles internally.
Stages where this happens are called out in their respective sections and discussed further there.
