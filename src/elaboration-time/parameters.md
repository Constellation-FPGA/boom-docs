# Elaboration-Time Parameters

Elaboration-time parameters are variables that must be provided at elaboration-/compile-time.
They must be passed from the upper/outer instantiating unit (These can simply be passed through too.).
You should never edit these directly, unless you konw what you are doing.
In most cases, these are already set by the BOOM core configuration.

* `coreWidth`: The number of micro-ops that may be committed simultaneously.
  This is what allows the core to be superscalar.
* `numLdqEntries`: The number of outstanding loads that are allowed to be queued for handling.
* `numStqEntries`: The number of outstanding stores that are allowed to be queued for handling.
