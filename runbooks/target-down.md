# Runbook — TargetDown

**Fires when:** Prometheus cannot scrape a target (`up == 0`) for 30 s. Severity `page`.

1. Which one? `up == 0` → label `job`.
2. Is the container running? `docker compose ps`, `docker compose logs --tail 50 <service>`.
3. Was it restarted / redeployed? `changes(process_start_time_seconds{job="<job>"}[1h])`.
4. Impact: if `job="payments"` every checkout fails with 502 "payments unreachable":
   `{service="shop"} | json | upstream_error != ""`.
5. Mitigate: `docker compose start <service>` (or `up -d <service>`). Verify `up{job="<job>"} == 1`
   and checkout error ratio back to 0.
