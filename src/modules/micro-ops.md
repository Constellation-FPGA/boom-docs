# Micro-ops

BOOM works in terms of micro-ops (\\( \mu \\)op, uop), **not** RISC-V instructions!

<div class="note">
Throughout this documentation, we will abbreviate micro-ops to <strong>uop</strong>.
</div>

Micro-ops are not a hardware module!
They cannot be elaborated or synthesized into hardware!
They are a class that describes control signals for the micro-op created out of a standard RISC-V instruction!

To enable true out-of-order execution, BOOM has both physical and architectural registers.
* Architectural registers are the ones defined by the specification and available to software.
  Programmers are free to manipulate these however they wish.
* Physical registers are ones that are implemented in the hardware.
  These are **not** addressable by the software programmer!
  Hardware uses these to provide the illusion/abstraction of in-order execution to software while actually executing out-of-order.

There are more (often significantly many more) physical registers than architectural registers.
For example, the default small BOOM V3 configuration has:
  * 52 integer physical registers, when the specification defines 31 (or 32, depending on how you count).
  * 48 floating-point physical registers, when the specification defines 32.

Below we have a list describing what some of the fields in the micro-op are:
* `pdst`: Index of the destination *physical* register the micro-op's result should be placed in.
* `prs1`: Index of the first source *physical* register the micro-op's result should be read from.
* `prs2`: Index of the second source *physical* register the micro-op's result should be read from, if it needs a second source.
* `prs3`: Index of the third source *physical* register the micro-op's result should be read from, if it needs a third source.
