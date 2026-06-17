# specs/ — declarative experiment & deployment specs

One spec describes one runnable configuration: which split is active
(monolithic / F1 / full O-RAN), the radio mode (ZMQ / RU emulator / real RU),
cell parameters, the scheduler under test, the workload, and the hypothesis it
serves. Validated at load, fail-closed — an unknown key or an unresolved
reference is an error, not a silent default.

Empty until the DSL and the first experiment are defined.
