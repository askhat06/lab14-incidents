# lab14-incidents — Observability: incidents, alerts, SLOs and logs

**Student:** Akylbek Askhat (ID s23068429) · GitHub [askhat06](https://github.com/askhat06)

**Variant: A** (Task 1 latency, Task 2 checkout SLO + burn-rate alerts, Task 3 noisy alert,
Task 4 request tracing with `request_id`, Task 5 dashboard as code) + **Final Challenge**.

- Report with all queries, results, screenshots and answers: [`REPORT.md`](REPORT.md)
- Postmortem of the reproduced incident: [`POSTMORTEM.md`](POSTMORTEM.md)

## Start the stack

Requirements: Docker (Desktop) with Compose v2. Nothing else — no cloud account, no Kubernetes.

```bash
docker compose up -d --build
docker compose ps                    # 9 services, all running
docker compose logs -f webhook       # alert notifications are printed here
```

| UI | URL |
|---|---|
| Prometheus | http://localhost:9090 |
| Alertmanager | http://localhost:9093 |
| Grafana (no login, local only) | http://localhost:3000 → Dashboards → lab14 → **ShopLab — RED** |
| Loki API | http://localhost:3100 |

Leave it running a few minutes: the traffic generator needs time to fill the graphs.

### The release

```bash
PAYMENTS_VERSION=1.1.0 docker compose up -d payments   # deploy 1.1.0
docker compose up -d payments                          # roll back to 1.0.0
```
(PowerShell: `$env:PAYMENTS_VERSION="1.1.0"; docker compose up -d payments; Remove-Item Env:PAYMENTS_VERSION`)

### Check and reload configuration

```bash
docker run --rm -v "$PWD/monitoring:/m" --entrypoint promtool prom/prometheus:v3.16.0 check rules /m/rules.yml
docker run --rm -v "$PWD/monitoring:/m" --entrypoint amtool prom/alertmanager:v0.35.0 check-config /m/alertmanager.yml
curl -X POST localhost:9090/-/reload
curl -X POST localhost:9093/-/reload
```

## Repository layout

```
compose.yaml                 the ShopLab stack (shop, payments, traffic, Prometheus, Alertmanager, webhook, Loki, Alloy, Grafana)
app/                         app.py (shop + payments), traffic.py, Dockerfile, requirements.txt
monitoring/
  prometheus.yml             scrape config
  rules.yml                  FINAL rules: TargetDown, HighErrorRatio (replaces ErrorsDetected), checkout SLO recording rules + burn-rate alerts
  alertmanager.yml           FINAL routing: group_by [alertname, job], severity=page -> /pager, rest -> /tickets
  rules.starter.yml          the inherited starter rules (kept for the Task 3 before/after comparison)
  alertmanager.starter.yml   the inherited starter routing (same reason)
  receiver.py                webhook stand-in for Slack/pager
  loki.yml, config.alloy     logs pipeline
  grafana/                   provisioned datasources + dashboard provider
dashboards/shoplab.json      "ShopLab — RED" dashboard (provisioned, loads on startup)
runbooks/                    checkout-slo.md, high-error-ratio.md, target-down.md
screenshots/                 evidence for REPORT.md
evidence/                    raw query output (text) and webhook logs used in the report
REPORT.md, POSTMORTEM.md
```

## Stop and clean up

```bash
docker compose down
docker image rm shoplab-app:1
```
