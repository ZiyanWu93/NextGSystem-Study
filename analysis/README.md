# analysis/ — reading the srsRAN code

What we learn from the system itself, organized along srsRAN's real seams.

| Folder | Holds |
|---|---|
| `architecture/` | the gNB disaggregation map: where CU / DU / RU are cut |
| `components/` | per-subsystem deep dives (CU-CP, CU-UP, DU-high, DU-low, RU) |
| `baseline/` | how to build and run it RF-free (ZMQ), plus the measured baseline |
| `bottlenecks/` | efficiency hotspots, with evidence |

`baseline/` comes **first**: an RF-free (ZMQ) bring-up is the foundation
everything else measures against. Empty until the baseline runs — that is the
correct state right now.
