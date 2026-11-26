# Benchmark & Observability Report — Day 14

> Số liệu CP7 lấy từ JSON trong `reports/`; số liệu scaling và observability CP8 lấy từ `benchmark-results/` và ảnh trong `screenshots/`.

**Họ tên:** Mai Danh Hieu
**Ngày chạy:** 26/11/2026
**Môi trường:** Windows PowerShell, Python 3.10 trong `.venv`; Intel Core i7-7700HQ (8 logical processors), RAM 15,8 GB; agent v1/v2 chạy local tại cổng 8001/8002.

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | ≤ 500 ms | Phát hiện phần request chậm mà trung bình dễ che khuất. Đây là ngưỡng mặc định của `load_test.py`. |
| Error rate `/ask` | ≤ 2% | Giới hạn lỗi upstream khi chịu tải; ngưỡng mặc định của `load_test.py`. |
| Accuracy trên golden dataset | ≥ 100% sau khi sửa KB | Gate cuối của CP7 yêu cầu cả 23 câu đúng và không regression. |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.01 | 208.6 | 263.4 | 1702.2 | 1.33% | PASS |
| 10 | 33.09 | 208.0 | 492.8 | 1714.3 | 1.67% | PASS |
| 50 | 38.57 | 563.2 | 3633.0 | 5664.0 | 1.33% | FAIL (p95) |
| 100 | 32.24 | 1870.6 | 6160.7 | 6789.6 | 1.33% | FAIL (p95) |

**Nhận xét:** RPS có tăng tuyến tính theo concurrency không? Vì sao?

RPS tăng mạnh từ concurrency 1 lên 10 nhưng không tăng tuyến tính: ở 50 chỉ đạt 38.57 RPS và ở 100 giảm còn 32.24 RPS. Khi nhiều request cùng chờ, latency và p95 tăng mạnh; thêm concurrency không tạo thêm năng lực xử lý tương ứng. Mỗi phép thử có 300 request; file nguồn là `reports/load_v2_c*.json`.

### 2.2 Tail latency

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 220.6 | 482.2 | 510.3 |
| 0.03 (mặc định) | 246.6 | 497.1 | 1687.1 |
| 0.1 | 354.4 | 1686.0 | 1718.8 |

**Nhận xét:** Chỉ số nào nhạy nhất với request chậm? Vì sao SLO không nên đặt theo average?

Các phép thử dùng 300 request, concurrency 20, chỉ đổi `MOCK_SLOW_RATE` giữa các lần khởi động v2. p99 nhạy nhất khi tỉ lệ request chậm từ 0 lên 0.03; đến 0.1, p95 cũng tăng mạnh và trượt SLO. Average chỉ tăng từ 220.6 lên 354.4 ms trong khi p95 tăng từ 482.2 lên 1686.0 ms, nên đặt SLO theo average sẽ bỏ qua trải nghiệm của nhóm người dùng chậm. Số liệu có yếu tố ngẫu nhiên từ mock LLM.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.35% (1/23) | 91.30% (21/23) | 100% (23/23) |
| Avg tokens / câu | 56.7 | 109.1 | 112.6 |
| Tổng cost (USD) | 0.000398 | 0.000762 | 0.000781 |
| Cost / câu đúng (USD) | 0.00039795 | 0.00003627 | 0.00003397 |
| Quality gate | FAIL (mặc định ≥ 80%) | PASS (mặc định ≥ 80%, không regression so với v1) | PASS (≥ 100%, không regression so với v2) |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` | `token`, `hết hạn`, `chữ ký` | Thêm entry kích hoạt bởi `jwt`/`api key` và giải thích JWT có chữ ký, thời hạn. |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Thêm entry kích hoạt bởi `kubernetes hpa`/`hpa` và nêu các chỉ số HPA. |

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**

v2 thêm retrieval nên accuracy tăng mạnh nhưng dùng nhiều token và tổng cost cao hơn v1. Tuy nhiên **cost trên mỗi câu đúng** giảm từ 0.00039795 xuống 0.00003627 USD; sau sửa KB còn 0.00003397 USD. Vì mục tiêu là câu trả lời đúng, cost/câu đúng hữu ích hơn cost/câu. Latency của eval 23 câu không đủ mẫu để kết luận hiệu năng; phần 2 dùng load test 300 request cho mỗi cấu hình. Chạy v1 với baseline v2 cho exit code 1 và 20 regression, chứng minh quality gate chặn bản giảm chất lượng.

---

## 4. Observability stack (production)

### 4.1 Bằng chứng

- [x] [Grafana dưới tải](screenshots/grafana-dashboard.png): traffic, error rate, p95, in-flight, 3 instances và cost.
- [x] [Jaeger trace chậm](screenshots/jaeger-slow-trace.png): POST `/ask` 2308.5 ms; span `chat mock-llm-small` 1705.4 ms.
- [x] [Jaeger trace lỗi](screenshots/jaeger-error-trace.png): POST `/ask` 503; span LLM và `agent.run` có event `exception` (xác nhận trong [trace JSON](benchmark-results/jaeger-error-trace.json)).
- [x] [Prometheus Alerts](screenshots/prometheus-alert-firing.png): `AgentHighErrorRate`, `AgentHighLatencyP95` và `LLMCostBurnRateHigh` cùng firing. [Ảnh chụp trạng thái API](benchmark-results/alerts_firing.json) ghi nhận thêm `AgentInstanceDown` firing.

Stack chạy bằng Docker Compose project `k5-day14-observability`; 3 agent, Nginx, Prometheus, Grafana, Jaeger và Locust đều lên. `promtool test rules prometheus/alert_rules_test.yml` trả `SUCCESS`. Locust chạy headless với `--no-deps` để Compose không đưa số agent về mặc định 2 replicas.

**Exercise 7.4 — đọc dashboard:** Ảnh Grafana ghi traffic **44.2 req/s**, error rate **0.65%**, p95 **547.95 ms** và **3 instances up** tại thời điểm chụp. Panel `Requests/s theo instance` dùng để kiểm tra Nginx có chia tải cho cả ba agent. Average thấp hơn p95 vì một nhóm request chậm (tail latency) kéo percentile lên; chỉ nhìn average sẽ bỏ qua nhóm này. Input tokens thường nhiều hơn output tokens vì prompt gửi LLM gồm cả câu hỏi, system prompt và context retrieval, còn câu trả lời ngắn hơn. Nginx trả **404** ở `/metrics` công khai để không lộ thông tin nội bộ; Prometheus scrape trực tiếp `/metrics` của từng agent trong mạng Compose.

### 4.2 Thí nghiệm “phá hệ thống”

Mốc bắt đầu A–C: **16:09:38 ngày 26/11/2026 (GMT+7)**, khi tạo lại 3 agent với `MOCK_ERROR_RATE=0.2`, `MOCK_SLOW_RATE=0.3`, `LLM_MODEL=mock-llm-large`, dưới tải Locust 100 users. Prometheus đánh giá mỗi 5 giây. Thời điểm firing được suy ra từ `activeAt + for` và kiểm tra lại qua API Alerts.

| # | Thay đổi | Alert firing | Thời gian từ lúc đổi → firing | Panel phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | `AgentHighErrorRate` | khoảng 84 giây (`for: 1m`) | Error rate, Requests/s theo status code |
| B | `MOCK_SLOW_RATE=0.3` | `AgentHighLatencyP95` | khoảng 139 giây (`for: 2m`) | Latency p95, biểu đồ p50/p95/p99 |
| C | `LLM_MODEL=mock-llm-large` | `LLMCostBurnRateHigh` đã firing trước khi đổi do tải baseline cũng vượt ngưỡng $2/giờ | Không đo được thời gian kích hoạt mới; alert đã firing tại mốc bắt đầu | Cost burn rate, LLM calls theo model |
| D | `docker pause` 1 trong 3 agent lúc 16:12:17 | `AgentInstanceDown` | khoảng 35–42 giây (`for: 30s`; quan sát API lúc 16:12:59) | Instances up, Prometheus Targets |
| E | `AGENT_VERSION=v1` | Không có alert chất lượng tương ứng | Không áp dụng | Quality gate trên golden dataset |

Với C, cost burn rate khoảng **$5.72/giờ** trước khi đổi và **$47.27/giờ** trong [snapshot bốn alert firing](benchmark-results/alerts_firing.json). Đây là thay đổi quan sát được dưới tải, nhưng không phải phép đo độc lập về thời gian firing của C vì alert đã hoạt động từ trước. Dùng lại cùng một mức tải thấp hơn và chờ alert về inactive nếu cần đo riêng thời gian của C.

Khi Nginx hết `proxy_read_timeout` và tự trả **504**, request có thể chưa kết thúc trong agent. Vì vậy counter `http_requests_total` của agent không nhất thiết ghi 504; cần theo dõi thêm log/metric 5xx tại Nginx để thấy đầy đủ lỗi ở phía người dùng.

**Phát hiện E trong production:** chạy eval định kỳ và trong CI trên golden dataset, so accuracy/câu regression với baseline đã duyệt. Trên bản v1 production, [eval report](benchmark-results/eval_v1_production.json) ghi **1/23 câu đúng (4.35%)**, 22 regression so với v2 23/23; quality gate trả exit code 1. Cần thêm alert chất lượng dựa trên eval theo lịch, vì traffic, error rate và latency không đo được độ đúng của câu trả lời.

### 4.3 Debug bằng trace

Trace chậm: [`03db16d361d8181eaf3f2a98a9e9c49d`](benchmark-results/jaeger-slow-trace.json), tổng **2308.5 ms**. Span `chat mock-llm-small` **1705.4 ms**, khoảng **73.9%** tổng thời gian; `agent.retrieve_context` 91.1 ms. Nút chậm chính nằm ở LLM mock, không phải retrieval. Log JSON cùng `trace_id`:

```json
{"ts":"2026-11-26T09:06:04.408341+00:00","level":"INFO","service":"observable-agent","instance":"11dd9a6d5134","event":"http_request","trace_id":"03db16d361d8181eaf3f2a98a9e9c49d","span_id":"d51d2d0a9734ec8c","request_id":"5bf600d18b88","method":"POST","route":"/ask","status":200,"ms":2202.8}
```

Trace lỗi: [`c2d75201c75e1a2b2683d2816b18c27a`](benchmark-results/jaeger-error-trace.json). Span `chat mock-llm-large` và `agent.run` đều ghi event `exception`; log JSON cùng `trace_id`:

```json
{"ts":"2026-11-26T09:10:35.529360+00:00","level":"INFO","service":"observable-agent","instance":"c52863f539ec","event":"llm_error","trace_id":"c2d75201c75e1a2b2683d2816b18c27a","span_id":"a45aff59ee77f28a","request_id":"43bc9d21f459","error":"Upstream LLM error: 503 Service Unavailable (mock)"}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, ramp 10/s, 2 phút)

Số liệu từ dòng `POST /ask` trong [scale_1_stats.csv](benchmark-results/scale_1_stats.csv) và [scale_3_stats.csv](benchmark-results/scale_3_stats.csv):

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---:|---:|---:|---:|---:|
| 1 | 54.21 | 220 | 1700 | 2600 | 66/6453 |
| 3 | 59.29 | 200 | 370 | 1700 | 67/7054 |

**Giải thích:** Scale từ 1 lên 3 làm p95 giảm khoảng **78.2%** và RPS tăng khoảng **9.4%**. Tail latency tốt hơn rõ rệt, nhưng throughput không tăng gấp ba vì Locust giữ cùng 100 users có thời gian nghỉ 0.5–2 giây, Nginx/host dùng chung tài nguyên, và latency mock LLM vẫn hiện diện. Failure chủ yếu là lỗi 503 mock (`MOCK_ERROR_RATE` mặc định 1%); lần 1 có một số 502 trong lúc agent mới khởi động. Kết quả là phép đo trên máy lab, không đại diện cho capacity production.

---

## 6. Kết luận

1. So RPS theo số user phải đọc cùng p95/p99 và tỷ lệ lỗi; thêm instance có thể cải thiện tail latency nhiều hơn throughput.
2. Metrics cho biết vấn đề xảy ra ở traffic, error, latency, saturation, cost; trace và log cùng `trace_id` chỉ ra span LLM chiếm thời gian hoặc ghi exception.
3. Alert cần ngưỡng và thời gian `for`; kiểm thử thật cho thấy độ trễ kích hoạt khoảng 84 giây (error), 139 giây (p95), 35–42 giây (instance down). Chất lượng câu trả lời cần eval riêng.

Nếu đưa lên production thật, thêm metric và alert cho **accuracy/regression trên golden dataset theo lịch**, tỷ lệ 429/timeout từ provider, queue wait time, tỷ lệ cache/retrieval hit và cost theo câu trả lời đúng. Dùng ngưỡng SLO theo traffic thực tế và kiểm tra lại trong môi trường staging.
