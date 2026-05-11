# Day 23 — Track 2 Observability Lab: Hướng dẫn chi tiết

> Dựa trên rubric (`rubric.md`) và nội dung các folder `01`–`05`.

---

## Tổng quan

Lab này xây dựng **một stack observability hoàn chỉnh** cho một AI inference service (mock LLM). Stack gồm 7 services chạy qua Docker Compose:

| Service | Container | Port | Vai trò |
|---|---|---|---|
| `app` | `day23-app` | 8000 | FastAPI mock LLM inference |
| `prometheus` | `day23-prometheus` | 9090 | Metrics storage + alerting |
| `alertmanager` | `day23-alertmanager` | 9093 | Alert routing → Slack |
| `grafana` | `day23-grafana` | 3000 | Dashboards (4 datasources) |
| `loki` | `day23-loki` | 3100 | Log aggregation |
| `jaeger` | `day23-jaeger` | 16686 | Distributed tracing UI |
| `otel-collector` | `day23-otel-collector` | 4317 / 4318 | OTLP receiver + tail-sampling |

**Quick start:**

```bash
make setup    # one-time: pull images, tạo .env
make up       # start 7 services
make smoke    # verify tất cả services đều OK
make demo     # end-to-end: load → alert → trace → drift
make verify   # rubric gate — exit 0 nếu tất cả checkpoints pass
make down     # stop stack
```

---

## Folder 01 — Instrument FastAPI AI Service

### Mục đích

Instrument một **mock LLM inference service** bằng FastAPI, phát ra cả 6 tín hiệu OTel:
- **Prometheus metrics** tại `/metrics`
- **OpenTelemetry traces** gửi qua OTLP gRPC tới `otel-collector:4317`
- **Structured JSON logs** via `structlog` → stdout → Promtail → Loki

### Endpoints

| Endpoint | Chức năng |
|---|---|
| `GET /healthz` | Liveness check — dùng cho Compose health-check |
| `POST /predict` | Mock LLM inference (random latency) |
| `GET /metrics` | Prometheus exposition |

### Metrics được instrument

| Metric | Type | Ý nghĩa |
|---|---|---|
| `inference_requests_total{model,status}` | Counter | RED: rate + errors |
| `inference_latency_seconds_bucket{model}` | Histogram | RED: duration |
| `inference_active_gauge` | Gauge | USE: in-flight requests |
| `gpu_utilization_percent` | Gauge | USE: GPU util (simulated) |
| `inference_tokens_total{model,direction}` | Counter | AI 4th pillar |
| `inference_quality_score{model}` | Gauge | Eval-as-metric stub |

### Cách chạy

```bash
# Cách 1: Chạy full stack (khuyến nghị)
make up

# Cách 2: Chạy standalone (không cần Docker)
cd 01-instrument-fastapi/app
pip install -r ../../requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8000
curl localhost:8000/metrics | head -40
```

### Theo rubric cần làm

| Checkpoint | Điểm | Cách verify |
|---|---|---|
| `/metrics` expose `inference_requests_total` | 5 | `curl localhost:8000/metrics \| grep inference_requests_total` |
| `/metrics` expose `inference_latency_seconds_bucket` | 5 | `curl localhost:8000/metrics \| grep inference_latency_seconds_bucket` |
| `inference_active_gauge` tăng khi có load, về 0 sau đó | 5 | Screenshot Grafana khi chạy `make load` |
| `inference_quality_score` và `inference_tokens_total` | 5 | `curl localhost:8000/metrics \| grep -E "inference_quality_score\|inference_tokens_total"` |

> **Tổng điểm folder 01: 20 điểm**

---

## Folder 02 — Prometheus, Grafana & Alertmanager

### Mục đích

Cấu hình **scraping metrics**, **dashboards-as-code**, và **SLO burn-rate alerting** cho AI service.

### Cấu trúc folder

```
02-prometheus-grafana/
├── prometheus/
│   ├── prometheus.yml          # Scrape config
│   └── rules/
│       ├── slo-burn-rate.yml   # Multi-window multi-burn-rate SLO
│       └── ai-quality.yml      # Latency, error rate, drift gauge alerts
├── alertmanager/
│   └── alertmanager.yml        # Severity routing → Slack
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/        # Auto-connect Prometheus, Loki, Jaeger
│   │   └── dashboards/         # Auto-provision dashboard JSONs
│   └── dashboards/
│       ├── ai-service-overview.json   # 6 panels: RPS, P50-95-99, errors, GPU, tokens, cost
│       ├── slo-burn-rate.json         # Error budget + burn rate
│       └── cost-and-tokens.json       # Token throughput + estimated cost
└── load-test/
    └── locustfile.py           # Load test scenarios
```

### 3 Dashboards được provision tự động

1. **AI Service Overview** — 6 panels: RPS, P50/P95/P99 latency, error rate, GPU util, tokens/sec, $/hr
2. **SLO Burn Rate** — Error budget remaining + multi-window burn rates
3. **Cost & Tokens** — Token throughput + estimated cost per hour

### Cách chạy

```bash
# 1. Mở Grafana UI
# http://localhost:3000 (admin/admin)
# Kiểm tra 3 dashboards đã load chưa

# 2. Chạy load test
make load
# Locust: 10 concurrent users, 60s, chạy baseline + spike + sustained-error scenarios

# 3. Kích hoạt alert
make alert
# Script sẽ kill app container → alert fire → restore → alert resolve

# 4. Xem Alertmanager
# http://localhost:9093

# 5. Kiểm tra Slack (nếu đã cấu hình webhook)
```

### Theo rubric cần làm

| Checkpoint | Điểm | Cách verify |
|---|---|---|
| 3 dashboards load tự động | 5 | Grafana API search — không cần import manual |
| Overview dashboard — 6 panels có data sau `make load` | 5 | Screenshot |
| SLO burn-rate dashboard có burn rates | 5 | Screenshot |
| Cost-and-tokens dashboard hiển thị $/hr khác 0 | 5 | Screenshot |
| `make alert` kích hoạt `ServiceDown` trong Alertmanager | 5 | Screenshot |
| Slack nhận được cả fire + resolve messages | 5 | Screenshot |

> **Tổng điểm folder 02: 30 điểm** (25 core + 5 bonus trong rubric tính gộp)

---

## Folder 03 — Distributed Tracing & Logs

### Mục đích

Xây dựng **distributed tracing** với OpenTelemetry + Jaeger, và **log aggregation** với Loki.

### Luồng dữ liệu

```
app → OTLP/gRPC (4317) → otel-collector → Jaeger (traces)
app → stdout JSON → otel-collector filelog receiver → Loki (logs)
```

### Tail-sampling policy

Composite policy trong `otel-collector/otel-config.yaml`:

1. **Keep 100%** traces với `status_code == ERROR`
2. **Keep 100%** traces với span duration > 2s
3. **Keep 1%** traces healthy (ngẫu nhiên)

Buffer: 30s decision window, ~10K spans memory.

### Span hierarchy cho `POST /predict`

```
POST /predict              (auto-instrumented FastAPI span)
├── embed-text             (manual span)
├── vector-search          (manual span)
└── generate-tokens       (manual span)
```

### Cách chạy

```bash
# Tạo 1 traced request
make trace
# curl POST /predict → in ra trace_id

# Mở Jaeger UI
# http://localhost:16686
# Search → Service: "inference-api" → Operation: "POST /predict"
# Click trace → xem flame graph với 3 child spans

# Mở Loki trong Grafana
# http://localhost:3000 → Explore → datasource: Loki
# Query: {service_name="inference-api"}
# Click trace_id trong log line → auto-link tới Jaeger

# Verify sampling policy
make alert  # kill app → request sẽ error → 100% được keep
# So sánh số lượng error traces vs healthy traces
```

### Theo rubric cần làm

| Checkpoint | Điểm | Cách verify |
|---|---|---|
| Jaeger UI có trace cho `POST /predict` với 3 child spans | 5 | Screenshot flame graph |
| Span attributes theo GenAI semantic conventions | 5 | Screenshot attributes panel |
| Tail-sampling: forced-error trace kept, healthy trace dropped (giải thích trong REFLECTION) | 5 | REFLECTION.md |
| Ít nhất 1 structured JSON log line có `trace_id` (paste trong REFLECTION) | 5 | REFLECTION.md |

> **Tổng điểm folder 03: 20 điểm**

---

## Folder 04 — Drift Detection (Evidently)

### Mục đích

So sánh phân phối **baseline** vs **current production** để phát hiện **data drift** và **prediction drift** bằng thư viện Evidently.

### Statistical tests được sử dụng

| Test | Full name | Phù hợp cho |
|---|---|---|
| **PSI** | Population Stability Index | Feature phân loại (categorical) — phổ biến trong banking/credit |
| **KL** | Kullback-Leibler Divergence | Feature phân loại (categorical) — đo "mất mát thông tin" |
| **KS** | Kolmogorov-Smirnov Test | Feature liên tục (numerical) — nhạy với shift ở trung tâm |
| **MMD** | Maximum Mean Discrepancy | Feature liên tục (numerical) — nhạy với shift ở đuôi phân phối |

### Cách chạy

```bash
cd 04-drift-detection
python3 scripts/drift_detect.py
# Output:
#   reports/drift-report.html    — Evidently HTML report
#   reports/drift-summary.json   — JSON summary (dùng cho rubric)
#   (optional) pushes drift_psi_score gauge lên Prometheus pushgateway
```

> **Lưu ý:** Đây là track duy nhất **không cần Docker** — có thể chạy trên Colab.

### Theo rubric cần làm

| Checkpoint | Điểm | Cách verify |
|---|---|---|
| `drift-summary.json` tồn tại và có ≥1 feature với `drift: yes` | 5 | Kiểm tra file content |
| Evidently HTML report render được | 5 | Screenshot |
| REFLECTION giải thích test nào (PSI/KL/KS/MMD) phù hợp với feature type nào | 5 | REFLECTION.md |

> **Tổng điểm folder 04: 15 điểm**

---

## Folder 05 — Integration với Days 16–22

### Mục đích

Kết nối observability stack với **các artifacts từ những ngày trước** (Phase 2 Track 2). Đây là "integrative day" — tích hợp mọi thứ đã học.

### Các nguồn dữ liệu tích hợp

| Ngày | Nguồn | Cách scrape |
|---|---|---|
| 16 — Cloud Infra | EC2/EKS hosts | `node_exporter` → thêm target vào `prometheus.yml` |
| 17 — Data Pipeline | Airflow DAG | `airflow_dag_run_duration` via `statsd_exporter` |
| 18 — Lakehouse | Spark / Delta | Spark UI metrics → Prometheus |
| 19 — Vector Store | Qdrant | Scrape `host.docker.internal:6333/metrics` |
| 20 — Model Serving | llama.cpp | Scrape `host.docker.internal:8080/metrics` |
| 21 — (skipped) | — | Chưa authored |
| 22 — Alignment | DPO model | Push `dpo_eval_pass_rate` gauge via script |

### Cách chạy

```bash
# Nếu có prior days chạy local:
# 1. Set env vars trong .env:
#    DAY19_QDRANT_URL=http://host.docker.internal:6333
#    DAY20_LLAMACPP_METRICS_URL=http://host.docker.internal:8080/metrics

# 2. Enable prometheus job stanzas (uncomment trong prometheus.yml)
# 3. Restart stack:
make restart

# Nếu KHÔNG có prior days:
# Các integration scripts sẽ tự động stub metrics
# Cross-day dashboard vẫn render với "No Data" panels
```

### Cross-day dashboard

File `05-integration/full-stack-dashboard.json` — 1 panel per source day (6 panels total). Được thiết kế fail-soft: panels không có data hiển thị "No Data" thay vì bị lỗi.

### Theo rubric cần làm

| Checkpoint | Điểm | Cách verify |
|---|---|---|
| Ít nhất 1 prior-day source được kết nối (real hoặc stub) | 5 | Screenshot cross-day dashboard |
| Cross-day dashboard render với đủ 6 panels (data hoặc "No Data") | 5 | Screenshot |

> **Tổng điểm folder 05: 10 điểm**

---

## Reflection (Folder submission/)

File `submission/REFLECTION.md` — yêu cầu bắt buộc, chiếm **15 điểm**.

### Các section cần có

1. **Section 1–5** — filled đầy đủ (5 điểm)
2. **"The single change that mattered most"** — paragraph có substance, không chỉ độ dài (10 điểm)

### Nội dung cần cover trong REFLECTION

- **03 tracing:** Tail-sampling policy — forced-error trace retained, healthy trace dropped (kèm math)
- **03 logs:** Paste ít nhất 1 structured JSON log line có `trace_id`
- **04 drift:** Giải thích PSI/KL/KS/MMD phù hợp với feature type nào
- **05 integration:** Mô tả prior-day metric nào khó expose nhất

---

## Bonus (20 pts)

| # | Bonus | Điểm | Yêu cầu |
|---|---|---|---|
| B1 | Pyroscope flame graph | +10 | Profiling Python process (Linux/WSL) |
| B2 | Langfuse self-hosted | +10 | Capture 1 LangChain LLM trace |

---

## Tổng kết rubric

| # | Folder | Checkpoints | Điểm |
|---|---|---|---|
| 1 | 00-setup | setup-report.json | 5 |
| 2 | 01-instrument-fastapi | Metrics (4 checkpoints) | 20 |
| 3 | 02-prometheus-grafana | Dashboards + Alerts (6 checkpoints) | 30 |
| 4 | 03-tracing-and-logs | Traces + Logs (4 checkpoints) | 20 |
| 5 | 04-drift-detection | Drift report + summary (3 checkpoints) | 15 |
| 6 | 05-integration | Cross-day dashboard (2 checkpoints) | 10 |
| 7 | REFLECTION.md | Sections + substance | 15 |
| **Core total** | | | **100** |
| B1–B2 | Bonus | Pyroscope + Langfuse | **+20** |

---

## Makefile Commands Cheat Sheet

| Command | Chức năng |
|---|---|
| `make setup` | Pull images + verify Docker |
| `make up` | Start stack (7 services) |
| `make down` | Stop stack |
| `make restart` | Stop + start |
| `make smoke` | Health-check tất cả services |
| `make load` | Run Locust load test (10 users, 60s) |
| `make alert` | Kill app → trigger alert → restore |
| `make trace` | Gửi 1 request + in trace_id |
| `make drift` | Run drift detection |
| `make demo` | End-to-end: load → alert → trace → drift |
| `make verify` | Rubric gate — exit 0 nếu pass |
| `make logs` | Tail logs từ tất cả services |
| `make clean` | Stop + xoá volumes (DESTRUCTIVE) |

---

## URL Reference

| Service | URL |
|---|---|
| FastAPI App | http://localhost:8000 |
| FastAPI /metrics | http://localhost:8000/metrics |
| FastAPI /docs | http://localhost:8000/docs |
| Prometheus | http://localhost:9090 |
| Alertmanager | http://localhost:9093 |
| Grafana | http://localhost:3000 (admin/admin) |
| Loki | http://localhost:3100 |
| Jaeger UI | http://localhost:16686 |
| OTel Collector metrics | http://localhost:8888/metrics |
