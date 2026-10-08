# bench-2 (LedgerLine) -- command reference

Every command used to build, run, break, observe and debug this test bench, in the order you need them.
Subject: **ledgerline** (gateway `http://<ip>:8081`). Repo: `https://github.com/Royson-salis-18/bench-2`.

## 1. Create the EC2 instance (AWS console)

| Setting | Value |
|---|---|
| AMI | Ubuntu Server 24.04 LTS |
| Type | `c7i-flex.large` (2 vCPU, 4 GiB) -- one bench needs ~0.6 GiB of containers; `t3.micro` is too small |
| Disk | 20 GiB gp3 |
| Key pair | create/download once (`.pem`); AWS cannot re-download it -- keep a copy |
| Security group inbound | TCP **22** from *My IP*, TCP **8081** from *My IP* (or your demo audience). Nothing else. |

## 2. Connect from your laptop (Windows PowerShell / cmd)

```
ssh -i "C:\Users\<you>\Downloads\<key>.pem" ubuntu@<instance-public-ip>
```
Use the **full path** to the key. "Permission denied (publickey)" = wrong key path or wrong user (`ubuntu` on Ubuntu, `ec2-user` on Amazon Linux).

## 3. Install and start the whole bench (one command, on the instance)

```
curl -fsSL https://raw.githubusercontent.com/Royson-salis-18/bench-2/main/bootstrap/ec2-setup.sh | bash
```
Installs Docker/git/make, clones the repo to `~/bench`, builds the images and runs `make bench` (application + traffic generator + Prometheus/Grafana/Loki/Promtail/docker-exporter). First run takes a few minutes. Log out and back in once so the `docker` group applies.

Options (environment variables before `bash`): `DEST=/path` (clone location), `BRANCH=main`, `GRAFANA_PASSWORD=...`, `DRY_RUN=1` (print only).

### Re-install / update to the latest repo
```
cd ~
sudo docker rm -f $(sudo docker ps -aq) 2>/dev/null
sudo rm -rf ~/bench
curl -fsSL https://raw.githubusercontent.com/Royson-salis-18/bench-2/main/bootstrap/ec2-setup.sh | bash
```
Quicker update without wiping: `cd ~/bench && git fetch --depth 1 origin main && git reset --hard FETCH_HEAD && cd ledgerline && sudo make down && sudo make bench`.
If apt says *"Could not get lock ... unattended-upgr"*, wait: `while sudo fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do sleep 5; done`.

## 4. Day-to-day (`cd ~/bench/ledgerline`)

| Command | What it does |
|---|---|
| `sudo make bench` | app + traffic + observability |
| `sudo make up` | app only (healthy baseline) |
| `sudo make test` | start if needed and run the API tests |
| `sudo make ps` | container status |
| `sudo make logs SVC=<service>` | follow one service's logs |
| `sudo make urls` | where to look |
| `sudo make down` | stop everything and delete volumes (databases re-seed next start) |
| `sudo make traffic-off` | stop the always-on traffic generator -- gateway goes quiet, only your own requests remain |
| `sudo make traffic-on` | start it again |
| `sudo make obs-up` / `obs-down` | observability only |
| `sudo make scenarios` | list production-issue scenarios |

`make scenario` and `make bench` start the traffic generator again -- run `traffic-off` afterwards if you want a quiet system.

## 5. Break it on purpose (scenarios)

```
cd ~/bench/ledgerline
sudo make scenarios                              # ids
sudo make scenario SCEN=ll-02-provider-outage     # apply one production condition
```
One command that applies a scenario, drives the load that triggers it, probes the symptoms, prints the **expected result** from `scenario.yaml` next to what happened, then resets (output also saved to `~/bench-results/`):
```
cd ~/bench
lab/bench-scenario.sh ledgerline ll-02-provider-outage 60          # 60 s of load
KEEP=1 lab/bench-scenario.sh ledgerline ll-02-provider-outage      # leave the scenario applied afterwards
```
Back to healthy: `sudo make scenario SCEN=ll-00-baseline`.
Full list with ground truth: [docs/SCENARIO-CATALOG.md](SCENARIO-CATALOG.md).

## 6. Drive traffic by hand

Quiet the bench, then call the API yourself:
```
cd ~/bench/ledgerline && sudo make traffic-off
```
The edge gateway is `http://<instance-ip>:8081` (`/api/accounts/...`, `/api/transfers`, `/api/statements/...`; see the service code under `ledgerline/services/` and the tests in `ledgerline/tests/` for request shapes).

Load generator directly (from the app directory):
```
sudo make load                                   # 20 users, 60 s
sudo docker run --rm --network ledgerline_default ledgerline/traffic:dev node loadgen/loadgen.mjs --base http://edge:8081 --concurrency 20 --duration 30
# add --continuous to run until stopped
```
Background-traffic tuning (env vars read by compose): `TRAFFIC_CONCURRENCY`, `TRAFFIC_THINK_MS`.

## 7. Observability (Grafana / Prometheus / Loki)

All monitoring listens on `127.0.0.1` of the instance only. From your laptop open a tunnel (leave it running):
```
ssh -i "C:\Users\<you>\Downloads\<key>.pem" -L 3000:localhost:3000 -L 9090:localhost:9090 ubuntu@<instance-public-ip>
```
Then: Grafana `http://localhost:3000` (admin / `bench`, anonymous viewing on; dashboard "test bench"), Prometheus `http://localhost:9090`.
Loki is queried through Grafana (Explore -> Loki, `{service="<name>"}`).

## 8. Point the mapper at it

Mapper UI -> *Add Project*: Target ID `ledgerline`, Host `<instance-public-ip>`, SSH user `ubuntu`, your `.pem` key.
The mapper discovers declared dependencies from the merged compose file at `~/bench/ledgerline/.rendered/docker-compose.yml`, observed connections from `/proc/net/tcp` inside containers, and metrics from `docker stats` / the gateway access log / Prometheus.

## 9. Debugging on the instance

```
sudo docker ps                                   # what is running / healthy
sudo docker ps -a                                # including stopped/crashed
sudo docker logs --tail 50 <container>           # e.g. ledgerline-transfers-1
sudo docker stats --no-stream                    # CPU / memory per container
sudo docker exec -it <container> sh              # shell inside
sudo docker compose -f ~/bench/ledgerline/.rendered/docker-compose.yml ps
curl -s http://localhost:8081/healthz          # gateway health
free -h; df -h /                                 # memory / disk
```

## 10. Common problems

| Symptom | Cause / fix |
|---|---|
| `No rule to make target 'traffic-off'` / `'bench'` | you are in an old clone, or the wrong folder -> update (section 3), `cd ~/bench/ledgerline` |
| Page does not load from your laptop | security group lacks TCP 8081 from your IP |
| `Permission denied (publickey)` | wrong key path or user |
| `rm: cannot remove ... .rendered/...: Permission denied` | files created by `sudo` -> `sudo rm -rf ~/bench` |
| Mapper shows no observed links for Grafana | an idle Grafana opens no connections until a dashboard is viewed (open it through the tunnel) |
| Mapper links red | stop traffic (`make traffic-off`), wait ~30 s; if still red a service is unhealthy (`sudo docker ps`) |
| Out of memory | use >= 4 GiB; swap is added automatically below 3 GiB |

## 11. Security hygiene

* Keep SSH (22) and the app port restricted to *My IP* when not demoing.
* The `.pem` never goes in GitHub; copy it between machines securely and rotate it if it was ever shared.
* Stop/terminate the instance when done to avoid charges.
