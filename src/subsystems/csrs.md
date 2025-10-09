# Control and Status Registers (CSRs)

BOOM reuses Rocket's CSR file for everything, including:
  * Tracking CSR values
  * Controlling reads & writes to privileged CSRs
  * Handling privilege mode changes due to traps
  * Determining which privilege mode a trap should go to

But, the two microarchitectures (BOOM and Rocket) are very different, so some compatibility shims are required in BOOM.

The most important one is that CSR instructions cannot be speculated.
This is done by stalling the entire pipeline until the CSR operation completes.
<div class="warning">
CSR instructions cannot be speculated.
</div>
