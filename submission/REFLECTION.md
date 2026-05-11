# Day 23 Lab Reflection

**Student:** Khổng Mạnh Tuấn 
**Submission date:** 2026-05-11
**Lab repo URL:** https://github.com/tuanjhg/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

`00-setup/setup-report.json`:

```json
{
  "docker": {"ok": true, "version": "29.4.0"},
  "compose_v2": {"ok": true, "version": "5.1.1"},
  "ram_gb_available": 9.56,
  "ram_ok": true,
  "required_ports": [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888],
  "bound_ports": [],
  "all_ports_free": true
}
```

The stack ran on Docker Desktop with WSL2 available. The Python host environment used Python 3.11.6 in `.venv`.

---

## 2. Track 02 - Dashboards & Alerts

Dashboard screenshots:

- `submission/screenshots/grafana-ai-service-overview.png`
- `submission/screenshots/grafana-slo-burn-rate.png`
- `submission/screenshots/grafana-cost-and-tokens.png`
- `submission/screenshots/grafana-cross-day-stack.png`

Load test result: Locust sent 1062 `POST /predict` requests with 0 failures. Approximate latency was p50 170 ms and p99 410 ms, with one cold/slow max around 2.3 s.

Alert drill:

| When | What | Evidence |
|---|---|---|
| T0 | stopped `day23-app` | Docker stop |
| T0+115s | `ServiceDown` fired | Alertmanager active alert |
| T1 | started `day23-app` | Docker start |
| T1+60s | `ServiceDown` resolved | Alertmanager returned no active alerts |

Slack note: Alertmanager is now configured to render `SLACK_WEBHOOK_URL` correctly from `.env`. A real Slack webhook still needs to replace the placeholder before Slack fire/resolve screenshots can be captured.

One thing that surprised me: the useful alert did not fire immediately after the container stopped. The scrape interval plus `for: 1m` means there is intentional delay, which is good operationally because it filters tiny restarts but also means demos need enough waiting time.

---

## 3. Track 03 - Tracing & Logs

Jaeger trace IDs:

- Forced-error trace retained by tail sampling: `f6a3fa4612250c5b8a3b824f9af12467`
- Slow successful trace with 3 child spans: `f4f4c1071deb0c1ddb60b4c48ba5b4e8`

The successful trace contains `predict` plus child spans `embed-text`, `vector-search`, and `generate-tokens`. The child spans reference `predict` with `CHILD_OF`, so the trace shows the inference path instead of four unrelated root spans.

Structured log line correlated to trace:

```json
{"model": "llama3-mock", "trace_id": "f6a3fa4612250c5b8a3b824f9af12467", "event": "forced failure", "level": "error", "timestamp": "2026-05-11T02:58:01.989353Z"}
```

Tail-sampling math: the collector policy keeps all error traces, all traces over 2000 ms, and 1% of healthy traces. If the service emits 20 traces/sec and they are all healthy/fast, expected retained traces are `20 * 0.01 = 0.2 traces/sec`. If 1 trace/sec is an error, retained traces become `1 + 19 * 0.01 = 1.19 traces/sec`.

---

## 4. Track 04 - Drift Detection

`04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {"psi": 3.461, "kl": 1.7977, "ks_stat": 0.702, "ks_pvalue": 0.0, "drift": "yes"},
  "embedding_norm": {"psi": 0.0191, "kl": 0.0322, "ks_stat": 0.052, "ks_pvalue": 0.133852, "drift": "no"},
  "response_length": {"psi": 0.0156, "kl": 0.0182, "ks_stat": 0.056, "ks_pvalue": 0.087838, "drift": "no"},
  "response_quality": {"psi": 8.8486, "kl": 13.5013, "ks_stat": 0.941, "ks_pvalue": 0.0, "drift": "yes"}
}
```

I would use PSI for `prompt_length` because it is easy to explain to product and ops stakeholders as distribution bucket movement. I would use KS for `embedding_norm` and `response_length` because they are continuous features and KS is sensitive to shape shifts without preselecting bins. For `response_quality`, I would track PSI for dashboard clarity and KS/KL for investigation because quality regression has both business impact and distribution-shape meaning. MMD would be my choice when comparing high-dimensional embeddings directly rather than a scalar norm.

---

## 5. Track 05 - Cross-Day Integration

The hardest prior-day metric to expose would be Day 18 Spark/Delta or Day 17 Airflow, because those systems usually have their own metrics surfaces, auth, and naming conventions. Qdrant and llama.cpp are simpler: they either expose Prometheus-compatible metrics or can be stubbed with a tiny monitor. The cross-day dashboard was provisioned as `day23-cross-day` and renders all six panels with data or explicit `No Data` states.

---

## 6. The single change that mattered most

The change that mattered most was making trace structure match the mental model of a request. Before that, the service could emit spans, but `predict`, `embed-text`, `vector-search`, and `generate-tokens` were not reliably connected as a useful tree. Once `predict` became the current parent span, the trace told an operator where time went inside the inference path. That is the difference between "we have tracing" and "we can debug a slow request."

The same idea appears in the deck's RED/USE framing: metrics tell me the service is slow or failing, but traces explain which internal step was responsible. I would rather have fewer, well-shaped spans with stable attributes than many unstructured spans that look impressive but do not shorten incident response.
