# bench-2 wiki: LedgerLine

An payments/ledger fintech microservice system built as a **test subject for the Microservice Mapper / RCA lab**. The code is correct; the failures come from production conditions (configuration drift, data volume, sizing, topology, third parties), so what you detect is what a real incident looks like.

## Contents
1. [Architecture](#architecture) - 2. [Repository map](#repository-map) - 3. [Run it](#run-it) - 4. [Use it as a customer](#use-it-as-a-customer) - 5. [Break it](#break-it) - 6. [Observe it](#observe-it) - 7. [Connect the mapper](#connect-the-mapper) - 8. [Troubleshooting](#troubleshooting) - 9. [Known limits](#known-limits)

## Architecture
```
 client / loadgen -> edge (nginx :8081, JSON access log with upstream_addr)
   /api/accounts   -> accounts :3101 -> state-store (Postgres) ... risk profile
   /api/transfers  -> transfers :3102 -> accounts, fraud-screening, fx-rates (-> fx-provider, circuit breaker), ledger (double-entry), state-store, event-bus
   /api/statements -> statements-worker :3106 -> ledger DB (heavy batch reads)
```
| Service | Role |
|---|---|
| edge | nginx gateway and Tier-1 access log |
| accounts | accounts and risk profiles |
| transfers | idempotent money movement (fee + fx + fraud screening + ledger) |
| ledger | double-entry postings (payer debited amount+fee, fee rolled up to treasury) |
| fraud-screening | synchronous risk check on the transfer path |
| fx-rates / fx-provider | FX quotes; third-party stand-in behind a circuit breaker |
| statements-worker | batch statement generation (memory heavy) |
| state-store, event-bus, ledger DB | Postgres x2, Redis, NATS |

## Repository map
| Path | What |
|---|---|
| `ledgerline/` | the application: `services/*`, `lib/` (shared HTTP, retry, breaker, NATS helpers), `nginx/`, `loadgen/` (journeys + organic sessions), `db/`, `tests/` |
| `ledgerline/scenarios/*/scenario.yaml` | six production conditions, each with ground truth, the load that triggers it, expected mapper signals and the fix |
| `ledgerline/docker-compose.yml` / `.obs.yml` | the app / the observability project (Prometheus, Grafana, Loki, Promtail, docker-exporter, grafana-viewer) |
| `ledgerline/Makefile` | `up test bench scenario traffic-off traffic-on obs-up obs-down down urls scenarios` |
| `lab/` | `bench-scenario.sh` (one-command scenario run + report), `chaos.sh` (fault injection on any service), `docker-exporter.js`, validators |
| `bootstrap/ec2-setup.sh` | one-command EC2 setup (Ubuntu / Amazon Linux) |
| `docs/` | this wiki, COMMANDS, CHAOS, SCENARIO-CATALOG, HOW-IT-FITS-THE-MAPPER |

## Run it
EC2: Ubuntu 24.04, `c7i-flex.large`, open TCP 22 and 8081 from your IP, then on the instance:
```
curl -fsSL https://raw.githubusercontent.com/Royson-salis-18/bench-2/main/bootstrap/ec2-setup.sh | bash
```
Locally (Docker): `cd ledgerline && make bench` (add `BUILD_CA_BUNDLE=/path/ca.crt` behind a TLS-intercepting proxy). Full step-by-step and updating: [COMMANDS.md](COMMANDS.md).

## Use it
There is no UI; call the API at `http://<ip>:8081/api/...` (see `ledgerline/services/` and `ledgerline/tests/` for request shapes) or let the always-on traffic run. `sudo make traffic-off` silences the background traffic; `traffic-on` restores it.

## Break it
* **Production-condition scenarios** (6: pool starvation cycle, provider outage, hot row, connection budget, batch OOM, shadow dependency): [SCENARIO-CATALOG.md](SCENARIO-CATALOG.md); run one with `lab/bench-scenario.sh ledgerline ll-02-provider-outage`.
* **Fault injection on any service** (`stop pause cpu net crash flap`) with predicted blast radius vs observed: `lab/chaos.sh` - [CHAOS.md](CHAOS.md).

## Observe it
Grafana `:3000` (dashboard "test bench", 25 panels), Prometheus `:9090`, Loki through Grafana. All on `127.0.0.1` of the instance; tunnel with `ssh -L 3000:localhost:3000 -L 9090:localhost:9090 ubuntu@<ip>`. 

## Connect the mapper
*Add Project*: Target ID `ledgerline`, Host = instance IP, user `ubuntu`, your `.pem`. The mapper reads declared edges from `ledgerline/.rendered/docker-compose.yml` (one merged file; multiple `-f` files break the compose label), observed edges from `/proc/net/tcp`, resources from `docker stats`, Tier 1 from the gateway log, Tier 2 from Prometheus. Details and findings about the mapper: [HOW-IT-FITS-THE-MAPPER.md](HOW-IT-FITS-THE-MAPPER.md).

## Troubleshooting
See the table in [COMMANDS.md](COMMANDS.md#10-common-problems). Most common: old clone on the instance (`git fetch --depth 1 origin main && git reset --hard FETCH_HEAD`), missing security-group rule for 8081, wrong key path.

## Known limits
Verified on Ubuntu 24.04 / c7i-flex.large and in a local Docker sandbox. The traffic generator can make mapper edges red under heavy scenarios; use `traffic-off` for a clean signal.
