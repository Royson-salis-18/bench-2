# Chaos and cascading-failure testing (LedgerLine)

The gateway (`edge`) is where users see every failure first, so a mapper's edge turns red there first. It is almost never the root cause.
`lab/chaos.sh` lets you break **any** service, watch the failure travel up the dependency graph, and compare it with a prediction, so you can check
that an RCA tool names the real root and that a cascade predictor gets the blast radius right.

```
cd ~/bench
lab/chaos.sh ledgerline list                              # services and their dependencies
lab/chaos.sh ledgerline predict ledger                     # predicted blast radius: who breaks if it breaks, hop by hop
lab/chaos.sh ledgerline run ledger pause 40                 # inject, sample every ~3 s, print the timeline and prediction vs observed, heal
lab/chaos.sh ledgerline inject fx-provider stop                 # leave a fault on while you look at the mapper
lab/chaos.sh ledgerline heal                              # undo every fault
```
Faults: `stop` (refused) - `pause` (accepts, never answers -> timeouts; the worst for cascades) - `cpu` (throttled to 5%, slow not dead) -
`net` (network partition) - `crash` (SIGKILL, restart policy revives it) - `flap` (frozen 4 s / running 4 s).
Each run is saved to `~/bench-results/chaos-*.txt`.

## Suggested cascade experiments
| Run | What you should see |
|---|---|
| `run ledger pause 40` | transfers fail (the ledger is the system of record); accounts reads keep working |
| `run fx-provider stop 40` | FX transfers fail fast once the breaker opens; same-currency transfers continue |
| `run fraud-screening pause 40` | transfers time out -> a cascade up to the edge |
| `run state-store crash 40` | rate/idempotency state lost briefly, then recovers |
| `run accounts cpu 40` | latency before errors (slow, not dead) |

## How to read the report
* **predicted blast radius**: everything that transitively `depends_on` the broken service (from the rendered compose the mapper also reads). It is the *worst case*.
* **gateway routes that failed**: what users actually saw (HTTP 5xx or timeout), with the second the first failure appeared.
* **predicted-but-healthy**: declared dependents that did not fail (the call is not on that route's hot path, or a cache/fallback absorbed it). Real systems are
  usually *smaller* than the graph; a good predictor should learn which declared edges are really on the hot path.
* **unexpected failures**: a route failed although its service is not downstream of the root. Should be empty; if not, a hidden dependency exists.
* A good RCA answer is the injected service, not the entry point.

Tip: `make traffic-off` first for a clean signal, or leave the always-on traffic running to see real error ratios in Grafana (dashboard "test bench") while the fault is on.
