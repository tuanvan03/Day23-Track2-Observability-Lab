# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** _Đoàn Văn Tuấn_
**Submission date:** _2026-05-11_
**Lab repo URL:** _https://github.com/tuanvan03/Day23-Track2-Observability-Lab_

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.2.1)
Compose v2:    OK  (5.0.2)
RAM available: 14.31 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: /Day23-Track2-Observability-Lab/00-setup/setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/02-img2.png`.

### Burn-rate panel

Drop `submission/screenshots/02-img6.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `02-img4.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `02-img3.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `02-img8.png` |

### One thing surprised me about Prometheus / Grafana

_(2-3 sentences)_

One thing that surprised me was how much configuration is needed to make dashboards "just work." Even with Grafana's auto-provisioning, I had to explicitly fix the datasource UID (`uid: prometheus`) in `datasources.yml` to match what the dashboard JSONs expected — without that, all panels showed "Datasource prometheus was not found." I also learned that `or vector(0)` is essential in recording rules to handle cases where no error metrics exist yet.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 19, "quality": 0.834, "duration_seconds": 0.2526, "trace_id": "96a9f24ae010861b49e7c4bbdcb5de8a", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T15:48:48.169037Z"}
```

**trace_id:** `96a9f24ae010861b49e7c4bbdcb5de8a`

### Tail-sampling math

If your service produced N traces/sec, what fraction did the policy keep? Show the calculation.

**Policy configuration (composite tail-sampling, 30s window):**

| Policy | Condition | Keep rate |
|---|---|---|
| keep-errors | `status_code == ERROR` | 100% |
| keep-slow | latency > 2000ms | 100% |
| probabilistic-1pct | random | 1% |

**Calculation (assuming N = 100 traces/sec, ~10% errors, ~5% slow):**
- Error traces kept: N × 10% × 100% = 10 traces/sec
- Slow-but-healthy traces kept: N × 5% × 100% = 5 traces/sec
- Remaining healthy traces: N × 85% = 85 traces/sec, of which 1% kept = 0.85
- **Total kept:** 10 + 5 + 0.85 ≈ 15.85 traces/sec (≈ 15.85% of all traces)

The math shows **error and slow traces dominate the sampling budget** — they are always kept, while only 1% of healthy traces survive. This means Jaeger's storage reflects the "interesting" traffic (problems + outliers) rather than the happy path.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

| Feature | Type | Best test | Why |
|---|---|---|---|
| `prompt_length` | Numerical (continuous) | **KS** (Kolmogorov-Smirnov) | KS is sensitive to shifts in the central tendency and distribution shape of continuous features. prompt_length's mean shifted from 50 → 85, which KS detects cleanly without binning artifacts. |
| `embedding_norm` | Numerical (continuous, bounded ~[0.7, 1.3]) | **MMD** (Maximum Mean Discrepancy) | Embedding norms are high-dimensional projections; MMD works well in reproducing-kernel Hilbert spaces and catches subtle distribution changes that binning-based tests (PSI/KL) might miss. |
| `response_length` | Numerical (continuous) | **KS** (Kolmogorov-Smirnov) | Like prompt_length, response_length is a straightforward continuous variable. KS is non-parametric and doesn't assume normality, making it robust for real-world LLM output lengths. |
| `response_quality` | Numerical (bounded [0, 1], Beta-distributed) | **PSI** (Population Stability Index) | PSI is the industry standard in production ML monitoring for scores/probabilities. It's less sensitive than KL to empty bins and works well with Beta-distributed quality scores that concentrate near 1.0. |

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

In my setup, no prior-day infrastructure was running, so I relied on stub scripts (`monitor-day19-vector-store.py` on :9101 and `monitor-day20-llama-cpp.py` on :9102) that emit fake Prometheus metrics. Getting Prometheus inside Docker to scrape the stubs running on the host required adding `extra_hosts: ["host.docker.internal:host-gateway"]` to the prometheus service, because `host.docker.internal` doesn't resolve on Linux by default. 

The **hardest metric to expose** would be Day 17's Airflow DAG duration, because Airflow doesn't export Prometheus metrics natively — you'd need a `statsd_exporter` sidecar plus custom Airflow config, which adds two more moving parts to debug. In contrast, the stub approach made Days 19 and 20 trivially testable.

---

## 6. The single change that mattered most

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.

The single change that made the biggest difference was **fixing the Grafana datasource UID to match what the dashboard JSONs expected**. Before this fix, all three provisioned dashboards rendered empty panels with a cryptic "Datasource prometheus was not found" error. The root cause was a subtle mismatch: the dashboards referenced `uid: "prometheus"` (lowercase), but Grafana's provisioning system auto-generated a different UID (`PBFA97CFB590B2093`). Adding `uid: prometheus` explicitly to the Prometheus datasource definition in `datasources.yml` was a one-line change that turned 15 empty panels into live, meaningful visualizations. This connects directly to the **Golden Signals / USE-RED methodology** from the deck: you can't debug latency spikes, error budgets, or saturation if your dashboards are broken before you even start. A working Overview dashboard with RPS, P99 latency, and error rate panels makes the difference between blindly restarting containers and confidently pinpointing root cause in seconds. No amount of sophisticated alerting matters if the foundation — a reliable datasource link — isn't in place.
