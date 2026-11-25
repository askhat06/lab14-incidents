# Runbook — HighErrorRatio (replaces the old `ErrorsDetected`)

**Fires when:** more than **2 %** of a service's requests return 5xx over the last 5 minutes, the service gets at
least 0.2 req/s (so 1 error out of 3 requests doesn't page), and this lasts **5 minutes**.
Labels: `job` (= service), `severity: page`.

Baseline: shop returns a random 500 on ~0.5 % of searches (≈0.35 % of all shop traffic) — this is known noise
and is deliberately below the threshold.

## 1. Confirm
```promql
sum by (job) (rate(http_requests_total{status=~"5.."}[5m])) / sum by (job) (rate(http_requests_total[5m]))
sum by (job, route, status) (rate(http_requests_total{status=~"5.."}[5m]))
```
## 2. Where / what
- `job="shop"` with `status="502"` on `/api/checkout` → upstream problem, go to payments (and `runbooks/checkout-slo.md`).
- `job="payments"` with `status="500"` → look at the error message and what failing requests share:
  ```logql
  {service="payments"} | json | level="ERROR" | msg != "request"
  sum by (card, version) (count_over_time({service="payments"} | json | status >= 500 [10m]))
  ```
- Did something change? `changes(process_start_time_seconds[1h])`, `docker compose ps`, deploy log.
## 3. Mitigate
Recent deploy → roll back (`docker compose up -d payments`). Otherwise escalate to the owning team with the queries above.
