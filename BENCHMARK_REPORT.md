# Benchmark & Observability Report — Day 14

> Số liệu CP7 lấy từ JSON trong `06-lab-complete/benchmark-observability/develop/reports/`. Phần observability và scaling production sẽ hoàn thành ở CP8.

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

### 4.1 Screenshot

- [ ] Dashboard Grafana khi đang có tải (thấy rõ Golden Signals)
- [ ] Một trace chậm trong Jaeger (thấy span nào chiếm thời gian)
- [ ] Một trace lỗi trong Jaeger (có event `exception`)
- [ ] Trang Alerts của Prometheus với ít nhất 1 alert đang **firing**

### 4.2 Thí nghiệm "phá hệ thống"

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | | | |
| B | `MOCK_SLOW_RATE=0.3` | | | |
| C | `LLM_MODEL=mock-llm-large` | | | |
| D | `docker pause` 1 instance | | | |
| E | `AGENT_VERSION=v1` | | | |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

...

### 4.3 Debug bằng trace

Chọn 1 trace chậm nhất bạn tìm được:

- `trace_id`: ...
- Tổng thời gian: ... ms
- Span chiếm nhiều thời gian nhất: ... (... ms, ...%)
- Dòng log tương ứng (grep theo `trace_id`):

```json

```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | | | | | |
| 3 | | | | | |

**Giải thích kết quả:** Scale có cải thiện p95 không? Nút thắt nằm ở đâu?

...

---

## 6. Kết luận

- 3 điều bạn học được về benchmark & observability:
  1. ...
  2. ...
  3. ...
- Nếu đưa agent này lên production thật, bạn sẽ thêm metric / alert nào nữa?

...
