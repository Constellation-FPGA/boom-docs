# FPU (Floating-Point Unit)

BOOM reuses Rocket's FPU, which itself uses [Berkeley's HardFloat](https://github.com/ucb-bar/berkeley-hardfloat) implementation.
Even though the FPU is lightly discussed in the [Execution Units](./execution-units.md) section, BOOM's FPU has additional material dedicated to it because a significant amount of engineering has gone into making Rocket's FPU work with BOOM.
