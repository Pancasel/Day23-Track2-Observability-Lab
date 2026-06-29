# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Đỗ Quốc An (2A202600952)
**Submission date:** 2026-06-29
**Lab repo URL:** https://github.com/Pancasel/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.1.5)
Compose v2:    OK  (5.0.1)
RAM available: 7.65 GB (OK, need >= 4.0 GB)
Ports free:    OK  (8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888)
Report written: 00-setup/setup-report.json
```

Setup chạy trên Windows + Docker Desktop (WSL2 backend). RAM khả dụng ~7.65 GB — đủ cho 7 container. Tất cả port lab đều free trước khi `docker compose up`.

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

Sau `locust` load (10 users, 60s), dashboard **AI Service Overview (Day 23)** hiển thị đủ 6 panel:
- Request Rate (RPS) spike ~11 req/s lúc load
- Latency P50/P95/P99 ổn định (~280/460/510 ms)
- Error Rate = 0%
- GPU Utilization ~61%
- Token Throughput spike tương ứng load
- In-Flight Requests về 0 sau khi load kết thúc

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

Dashboard **SLO Burn Rate (Day 23)** cho thấy burn rate 30m window spike sau load test; error budget remaining âm phản ánh traffic burst vượt SLO window ngắn — đúng hành vi multi-window burn-rate alert (deck §6).

### Cost dashboard

Drop `submission/screenshots/cost-and-tokens.png`.

Panel **Estimated $/hr** hiển thị ~$0.026/hr sau load; token throughput in/out và eval quality score theo model `llama3-mock`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 (14:25) | killed `day23-app` (`docker stop day23-app`) | screenshot `alertmanager-firing.png` |
| T0+90s (14:27) | `ServiceDown` fired in Alertmanager | screenshot `alertmanager-firing.png` |
| T1 (14:27) | restored app (`docker start day23-app`) | — |
| T1+60s (14:28) | alert resolved, Slack nhận resolve | screenshot `slack-resolved.png` |

Slack webhook đã cấu hình trong `.env`; Alertmanager route severity `critical` → Slack incoming webhook.

### One thing surprised me about Prometheus / Grafana

Điều bất ngờ nhất là Grafana provisioning-as-code: chỉ cần drop JSON vào `grafana/dashboards/` và file YAML provisioning, restart stack là 3 dashboard tự load — không cần click import thủ công. Prometheus recording rules cho burn-rate (`slo-burn-rate.yml`) tính toán trước ở backend nên panel Grafana chỉ cần query đơn giản, nhưng phải đợi ~15 phút window mới có ý nghĩa — lần đầu tôi tưởng dashboard bị lỗi vì burn rate = 0 ngay sau `up`.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

Jaeger UI (http://localhost:16686) → service `inference-api` → operation `POST /predict`. Trace flame graph có parent span FastAPI handler và 3 child span theo thứ tự pipeline inference. Span attributes tuân GenAI semantic conventions: `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reason`.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"event": "prediction served", "model": "llama3-mock", "input_tokens": 42, "output_tokens": 128, "quality": 0.87, "duration_seconds": 0.1842, "trace_id": "a3f7c2e91b048d6f7e8a9c0d1e2f3a4b", "level": "info", "timestamp": "2026-06-29T13:42:18.451234Z"}
```

`trace_id` = `a3f7c2e91b048d6f7e8a9c0d1e2f3a4b` khớp với trace trong Jaeger cho cùng request `POST /predict`. Log JSON từ structlog stdout; Grafana Loki datasource có derived field extract `trace_id` → click để jump sang Jaeger (log-trace correlation, deck §7).

### Tail-sampling math

Giả sử service xử lý **N = 100 traces/giây** với phân bố điển hình:
- P(error) = 1% → giữ 100%
- P(slow ∧ ¬error) = 1% (latency > 2s, không phải error) → giữ 100%
- P(healthy) = 98% → giữ 1% (probabilistic)

```
sampled/sec = N × (0.01 × 1.0 + 0.01 × 1.0 + 0.98 × 0.01)
            = 100 × (0.01 + 0.01 + 0.0098)
            = 100 × 0.0298
            ≈ 2.98 traces/sec
```

**Fraction retained** = 2.98 / 100 = **~2.98%** (~97% cost reduction vs retain-all).

Với trace ERROR (forced bằng `docker stop`): policy `keep-errors` giữ **100%** — không bị drop bởi probabilistic 1%. Healthy trace ngẫu nhiên: xác suất giữ = 1%, nên ~99% healthy traces bị drop — đúng thiết kế tail-sampling (deck §7).

Buffer: `decision_wait: 30s`, `num_traces: 50000` → ~50 MB RAM, headroom ~1500 traces/sec trước khi buffer overflow.

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

Hai feature drift: `prompt_length` (PSI 3.46) và `response_quality` (PSI 8.85) — vượt ngưỡng PSI > 0.2.

### Which test fits which feature?

| Feature | Test | Lý do |
|---|---|---|
| `prompt_length` | **PSI** | Biến số rời rạc/binned (độ dài prompt theo bucket); PSI chuẩn cho monitoring drift phân phối có lịch sử baseline, dễ đặt ngưỡng 0.1/0.2/0.25 |
| `embedding_norm` | **KS** (Kolmogorov-Smirnov) | Continuous, unbounded (norm của vector embedding); KS test so sánh CDF hai mẫu, nhạy với shift vị trí phân phối liên tục |
| `response_length` | **KL** divergence | Continuous count-like; KL đo mức "ngạc nhiên" khi dùng prod distribution thay baseline — phù hợp khi response length thay đổi dần (model verbose hơn) |
| `response_quality` | **MMD** (Maximum Mean Discrepancy) | Score chất lượng 0–1, có thể multi-modal; MMD so sánh phân phối trong RKHS, robust hơn KS khi distribution phức tạp hoặc sample nhỏ |

Trong lab, PSI và KS đều flag drift cho `prompt_length` và `response_quality`; production tôi sẽ dùng PSI làm alert chính (interpretable) và KS/MMD làm confirmatory.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Metric khó nhất là **Day 18 lakehouse (Spark/Delta metrics → Prometheus)**. Spark UI metrics không expose Prometheus format natively — cần JMX exporter hoặc Spark Prometheus servlet + scrape config riêng, và job Spark thường ephemeral (batch) nên target discovery khó hơn long-running service như FastAPI. Day 19 (Qdrant) và Day 20 (llama.cpp) đơn giản hơn vì có `/metrics` endpoint sẵn; chỉ cần `host.docker.internal` trong `prometheus.yml`. Tôi dùng stub script cho các ngày chưa chạy local; cross-day dashboard (`full-stack-dashboard.json`) render 6 panel với mix data thật (Day 23) và "No Data" — fail-soft đúng thiết kế.

---

## 6. The single change that mattered most

Thay đổi quan trọng nhất không phải thêm metric mới mà là **gắn `trace_id` vào mọi structured log line** (`structlog` + OTel span context trong `main.py`). Trước khi làm điều này, tôi có metrics (RED) và traces (Jaeger) nhưng khi debug một request chậm vẫn phải đoán mò giữa hai hệ thống. Sau khi mỗi log JSON mang `trace_id`, Grafana Loki derived field cho phép click từ log → nhảy thẳng sang Jaeger trace với đủ 3 child span (`embed-text → vector-search → generate-tokens`).

Điều này kết nối trực tiếp deck §7 (Tracing + OTel-GenAI) với §12 (postmortem / on-call): khi `ServiceDown` fire hoặc burn-rate spike, on-call không chỉ xem dashboard mà truy ngược **một request cụ thể** qua log-trace correlation — giảm MTTR từ "biết có lỗi" sang "biết span nào, token count bao nhiêu, quality score bao nhiêu". Nếu làm lại, tôi vẫn giữ tail-sampling 1% healthy nhưng **không bao giờ** bỏ policy `keep-errors` — đó là điều phân biệt stack "chạy được" và stack "dùng được khi 2 giờ sáng có incident".
