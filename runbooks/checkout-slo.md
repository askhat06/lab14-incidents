# Runbook — Checkout availability SLO (CheckoutErrorBudgetFastBurn / CheckoutErrorBudgetSlowBurn)

**SLO:** 99.5 % of `POST /api/checkout` requests succeed (non-5xx), measured over 30 days.
**Error budget:** 0.5 % of checkouts = 216 minutes of full outage per 30 days.

| Alert | Condition | Severity | Meaning |
|---|---|---|---|
| `CheckoutErrorBudgetFastBurn` | burn rate > 14.4× over **1h and 5m** | `page` | 2 % of the monthly budget gone in 1 h; budget empty in ~2 days. Act now. |
| `CheckoutErrorBudgetSlowBurn` | burn rate > 6× over **6h and 30m** | `ticket` | 5 % of the budget gone in 6 h; budget empty in ~5 days. Fix during working hours. |

burn rate = checkout error ratio / 0.005

## 1. Confirm (2 min)
```promql
slo:checkout_error_ratio:rate5m            # is it still happening?
slo:checkout_error_ratio:rate1h / 0.005    # burn rate
sum by (job, route, status) (rate(http_requests_total{status=~"5.."}[5m]))   # who returns 5xx?
```
Open Grafana → **lab14 / ShopLab — RED**, panel *Checkout error budget: burn rate*.

## 2. Find the cause
- shop returns **502** = the upstream (payments) failed or is unreachable. Look at payments:
  ```promql
  sum by (status) (rate(http_requests_total{job="payments"}[5m]))
  up{job="payments"}
  changes(process_start_time_seconds{job="payments"}[1h])    # recent deploy/restart?
  ```
- What do failures have in common?
  ```logql
  sum by (card) (count_over_time({service="payments"} | json | status >= 500 [10m]))
  {service="payments"} | json | level="ERROR" | msg != "request"
  {service="shop"} | json | status=502 | line_format "{{.request_id}} {{.card}} {{.upstream_status}}{{.upstream_error}}"
  ```
- Follow one request across services: `{service=~"shop|payments"} |= "<request_id>"`

## 3. Mitigate (first stop the bleeding, then debug)
- Error started right after a payments deploy → **roll back**:
  `docker compose up -d payments` (back to 1.0.0). Note the time (UTC) for the postmortem.
- payments down (`TargetDown{job="payments"}`) → `docker compose start payments`, check `docker compose logs payments`.

## 4. Verify
`slo:checkout_error_ratio:rate5m` back to ~0 within 5 min; alert resolves (the 1h window keeps the fast-burn
`for` condition true a bit longer — that is expected).

## 5. Afterwards
Write down how much budget was used: `increase(http_requests_total{job="shop",route="/api/checkout",status=~"5.."}[<incident duration>])`
and start a blameless postmortem if > 5 % of the monthly budget was spent.
