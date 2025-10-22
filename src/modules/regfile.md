# Register File

Boom implements a custom register file class so that it can multi-port the register file.

## Registers Files in BOOM
There are three major register files in BOOM:

  1. Integer register file (`iregfile` in `core.scala`)
  2. Floating-point register file (`fregfile` in `fp-pipeline.scala`)
  3. Predicate register file (`pregfile` in `core.scala`)

### Integer Register File (`iregfile`)
The integer register file is exactly what you would expect from the RISC-V specification.
32 addressable registers with 31 modifiable registers with the zeroth register always being hard-wired to zero.

### Floating-point Register File (`fregfile`)
The floating-point register file is only included in the core if floating-point support is enabled.
This is quite similar to the integer register file, but creates 32 registers that are each `fLen` wide.

### Predicate Register File (`pregfile`)
The predicate register file is unique among the other register files because every entry in it is just one (1) bit wide!
The generation of this register file is controlled by the [`enableSFBOpt` flag](../elaboration-time/parameters.md#boomcoreparams).
