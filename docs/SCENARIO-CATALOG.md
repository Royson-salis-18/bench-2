# Scenario catalogue

Generated from each `scenario.yaml`; open one for why review/tests pass, the production condition, the load that triggers it, what a mapper should see, its blind spots and the fix.

| Scenario | What happens | Class | Verified | True root cause |
|---|---|---|---|---|
| [`ll-00-baseline`](ledgerline/scenarios/ll-00-baseline/scenario.yaml) | Healthy baseline (control group) | control | — | — |
| [`ll-01-pool-starvation-cycle`](ledgerline/scenarios/ll-01-pool-starvation-cycle/scenario.yaml) | Two healthy services wait on each other and nothing looks busy | circular dependency / resource starvation | local-process | accounts <-> fraud-screening (cycle) |
| [`ll-02-provider-outage`](ledgerline/scenarios/ll-02-provider-outage/scenario.yaml) | Third-party outage with a circuit breaker -- the silence is the symptom | external dependency failure / partial outage | local-process | fx-provider |
| [`ll-03-hot-row`](ledgerline/scenarios/ll-03-hot-row/scenario.yaml) | Every transfer in the bank queues on one row | lock contention / data skew | local-process | ledger-db |
| [`ll-04-connection-budget`](ledgerline/scenarios/ll-04-connection-budget/scenario.yaml) | Pools that are each reasonable add up to more than the database allows | capacity planning / connection exhaustion | local-process (exhaustion reproduced; the restart/failover amplification is documented but NOT reproduced locally) | ledger-db |
| [`ll-05-batch-oom`](ledgerline/scenarios/ll-05-batch-oom/scenario.yaml) | The month-end job that kills itself, restarts, and tries again | data skew / batch memory / restart loop | docker + local-process | statements-worker |
| [`ll-06-shadow-dependency`](ledgerline/scenarios/ll-06-shadow-dependency/scenario.yaml) | fraud-screening quietly depends on fx-rates | undocumented dependency | local-process (flag path) | — |
