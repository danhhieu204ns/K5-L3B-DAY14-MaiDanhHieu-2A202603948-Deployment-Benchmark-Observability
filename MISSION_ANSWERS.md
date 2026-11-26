# Day 14 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found

1. **Hardcode thông tin nhạy cảm:** `OPENAI_API_KEY` và `DATABASE_URL` (chứa mật khẩu) được viết trực tiếp trong `develop/app.py` dòng 17–18. Nếu đẩy mã nguồn lên repository công khai, thông tin này có thể bị lộ.
2. **Không quản lý cấu hình theo môi trường:** `DEBUG = True` và `MAX_TOKENS = 500` cố định ở dòng 21–22. Muốn đổi cấu hình cho staging/production phải sửa code và triển khai lại.
3. **Dùng `print()` và ghi cả secret vào log:** Hàm `ask_agent()` ở dòng 33–38 in câu hỏi, câu trả lời và API key. Log khó xử lý tự động, đồng thời có nguy cơ lộ key.
4. **Không có endpoint health check:** Bản develop chỉ khai báo `/` và `/ask` (dòng 25–31), không có `/health`. Nền tảng triển khai không có endpoint riêng để kiểm tra process còn hoạt động.
5. **Host và port cố định:** `uvicorn.run()` đặt `host="localhost"`, `port=8000` ở dòng 51–52. App khó nhận kết nối từ ngoài container và không dùng được cổng do nền tảng cloud cấp qua biến `PORT`.
6. **Luôn bật chế độ reload:** `reload=True` ở dòng 53 phục vụ phát triển, nhưng tạo thêm process theo dõi thay đổi file và không phù hợp để chạy production. File cũng không có logic dọn tài nguyên khi shutdown.

### Exercise 1.2: Run develop version

Lệnh `POST /ask?question=Hello` trả `HTTP/1.1 200 OK` và JSON có trường `answer` (kết quả chạy ngày 26/11/2026). Điều này xác nhận ví dụ chạy được trên máy local, chưa chứng minh nó sẵn sàng cho production.

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Tại sao quan trọng? |
|---|---|---|---|
| Cấu hình và secret | API key, URL database, debug và port viết trực tiếp trong `app.py`. | `config.py` đọc cấu hình từ biến môi trường. | Đổi cấu hình theo môi trường triển khai mà không sửa code; tránh đưa secret vào repository. |
| Health và readiness | Không có `/health` hoặc `/ready`. | Có `GET /health` và `GET /ready`. | Nền tảng biết process còn sống và khi nào app sẵn sàng nhận request. |
| Logging | Dùng `print()`, kể cả in API key. | Dùng `logging` và ghi sự kiện request/response mà không ghi key. | Dễ theo dõi sự cố và giảm nguy cơ lộ secret trong log. |
| Shutdown | Không có xử lý vòng đời ứng dụng. | Có `lifespan` để thực hiện bước shutdown. | Có chỗ dọn tài nguyên và xử lý khi dừng service. |
| Host, port và reload | `localhost`, cổng `8000`, `reload=True` cố định. | Host/port/debug lấy từ cấu hình; mặc định host `0.0.0.0`. | Nhận kết nối trong container và dùng cổng được cấp trên cloud; chỉ bật reload khi cần phát triển. |
| Dữ liệu gửi tới `/ask` | `question` qua query string. | `question` trong JSON body. | Client cần gửi đúng định dạng mà từng phiên bản yêu cầu. |

**Kết quả chạy thực tế:** bản production trả `200 OK` cho `/health` với `status: ok` và cho `/ready` với `ready: true`. Lệnh POST JSON bằng `Invoke-RestMethod` trả câu trả lời mock cho `question: Hello`. 

## Part 2: Docker

### Exercise 2.1: Dockerfile cơ bản

1. **Base image:** `python:3.11`, chứa Python 3.11 và các thành phần cần thiết để chạy ứng dụng.
2. **Working directory:** `/app` trong container (`WORKDIR /app`). Các lệnh tiếp theo chạy tương đối với thư mục này.
3. **Vì sao COPY `requirements.txt` trước:** Docker có thể tái sử dụng layer cài dependency khi chỉ mã nguồn ứng dụng thay đổi; build lại nhanh hơn.
4. **CMD và ENTRYPOINT:** `CMD` đặt lệnh hoặc tham số mặc định, có thể thay bằng lệnh ở `docker run`. `ENTRYPOINT` đặt chương trình chính của container; đối số thêm ở `docker run` thường được truyền cho chương trình đó. Có thể dùng cả hai: `ENTRYPOINT` là chương trình, `CMD` là tham số mặc định.

### Exercise 2.2: Build và chạy image develop

Build từ gốc repo: `docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .` thành công. Vì cổng 8000 trên máy đang được app CP1 dùng, container được thử với `docker run -d --name k5-day14-docker-develop -p 8001:8000 my-agent:develop`. Gọi `POST http://localhost:8001/ask?question=What%20is%20Docker%3F` trả `HTTP/1.1 200 OK` và câu trả lời mock: “Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!”. Đã dừng và xóa container thử nghiệm.

**Lưu ý:** app develop của Part 2 nhận `question` qua query string; lệnh JSON body trong `CODE_LAB.md` không khớp với code này.

### Exercise 2.3: Multi-stage build và kích thước image

Stage `builder` cài công cụ build và Python dependencies. Stage `runtime` bắt đầu từ `python:3.11-slim`, chỉ chép dependencies đã cài và mã nguồn cần chạy, tạo user không phải root. Công cụ biên dịch trong builder không đi vào image cuối, nên image runtime dự kiến nhỏ hơn bản develop dùng `python:3.11` đầy đủ.

Build từ gốc repo: `docker build -f 02-docker/production/Dockerfile -t my-agent:advanced .` thành công. Kết quả `docker images my-agent` trên máy này:

| Image | Kích thước hiển thị |
|---|---:|
| `my-agent:develop` | 1,67 GB (xấp xỉ 1670 MB) |
| `my-agent:advanced` | 240 MB |

Image advanced nhỏ hơn khoảng `(1670 - 240) / 1670 × 100 = 85,6%`. Đây là số hiển thị của `docker images`, có làm tròn.

### Exercise 2.4: Docker Compose stack

Compose khởi động 4 service: `nginx`, `agent`, `redis`, `qdrant`. `nginx` là cổng HTTP công khai và chuyển request đến `agent:8000` trong mạng `internal`. Agent nhận địa chỉ `REDIS_URL=redis://redis:6379/0` và `QDRANT_URL=http://qdrant:6333`; Compose chờ Redis và Qdrant healthy trước khi khởi động agent. Redis lưu cache/session; Qdrant là vector database. Dữ liệu của hai service được gắn vào Docker volumes.

```text
Client → Nginx (:80) → Agent (:8000)
                         ├→ Redis (:6379)
                         └→ Qdrant (:6333)
```

Đã chạy `docker compose -p k5-day14-docker -f 02-docker/production/docker-compose.yml up -d --build`. Qua Nginx, `GET http://localhost/health` trả `HTTP/1.1 200 OK` với `status: ok`; `POST http://localhost/ask` với JSON `{"question":"Explain microservices"}` trả status `200` và trường `answer`. Tạo file `.env.local` rỗng (được `.gitignore` bỏ qua) vì Compose yêu cầu file này.

## Part 3: Cloud Deployment (CP6)

### Exercise 3.2: Render deployment

Public URL: https://k5-l3b-day14-maidanhhieu-2a202603948.onrender.com

Render Web Service dùng Docker, branch `main`, root directory `06-lab-complete/`. Dashboard báo **Deploy succeeded · Live** cho commit `51017d1` ngày 26/11/2026. Kiểm tra URL public: `GET /health` → **200**, `GET /ready` → **200**, `POST /ask` thiếu `X-API-Key` → **401**, `POST /ask` có key hợp lệ → **200** với JSON có `question`, `answer`, `cost_usd`. `GET /` → 404 vì không có route gốc. Không ghi key vào repo. Thông tin và lệnh kiểm tra: `DEPLOYMENT.md`.

### So sánh Render với Railway

`railway.toml` chọn Dockerfile builder và `healthcheckPath = "/health"`; Railway cấu hình root directory và biến môi trường của service riêng trên dashboard. `06-lab-complete/render.yaml` mô tả web service Docker, `rootDir`, `healthCheckPath: /health` và danh sách env vars; secret dùng `sync: false` để nhập trên dashboard. Điểm khác là Railway dùng TOML cho build/deploy của service, còn Render Blueprint dùng YAML để khai báo service và biến môi trường. Web service hiện tại được tạo thủ công trên Render; file `render.yaml` trong thư mục con là template tham khảo, chưa được Render tự áp dụng. Nếu dùng Blueprint, cần đưa bản sao file ra gốc repo.

### Exercise 3.3: Cloud Run CI/CD

`cloudbuild.yaml` mô tả pipeline: cài dependency và chạy pytest → build Docker image gắn commit SHA → push các tag lên registry → deploy image lên Cloud Run ở `asia-southeast1`. `service.yaml` định nghĩa service với giới hạn CPU/RAM, concurrency, min/max instances, secret từ Secret Manager và các probe `/health`, `/ready`. Đây là phân tích cấu hình mẫu; không có deployment Cloud Run trong bài này.

## Part 4: API Security (CP3)

### Exercise 4.1: API key authentication

`develop/app.py` lấy key từ `AGENT_API_KEY` và kiểm tra header `X-API-Key` trong dependency `verify_api_key` của `POST /ask`. Thiếu key trả **401**, key sai trả **403**, key đúng trả **200**. Muốn rotate key, đổi biến môi trường `AGENT_API_KEY` trên service rồi khởi động lại app; client phải dùng key mới. Bản demo in key lúc khởi động, vì vậy không nên dùng cách log đó với secret thật.

Kết quả chạy trên cổng 8003: không có key → 401; key sai → 403; key hợp lệ → 200 và JSON có `question`, `answer`. Lần gửi JSON bằng `curl.exe` trong PowerShell trả 500 vì body bị sai định dạng; traceback là `JSONDecodeError`. Gửi cùng body bằng `Invoke-WebRequest` trả 200.

### Exercise 4.2: JWT authentication

`POST /auth/token` với tài khoản demo `student`/`demo123` trả JWT bearer, hạn 60 phút. Client gửi token trong header `Authorization: Bearer <token>`; `verify_token` kiểm tra chữ ký, thời hạn rồi lấy `sub` và `role`. Thiếu token → **401**. Token không được ghi vào báo cáo.

### Exercise 4.3: Rate limiting

`rate_limiter.py` dùng **sliding window**: lưu timestamp request theo từng username trong deque, bỏ timestamp cũ hơn 60 giây. User thường có giới hạn **10 request/60 giây**, admin **100 request/60 giây** (tier riêng, không phải vô hạn). Với token `student`, 10 request đầu trả **200**; request thứ 11 và 12 trả **429**, có header `Retry-After`. Dữ liệu giới hạn hiện nằm trong RAM nên sẽ mất khi restart hoặc chạy nhiều instance; triển khai thật nên dùng Redis.

Output `curl.exe -i` rút gọn từ lần chạy thực tế (không chứa token):

```text
POST /ask, không có Authorization
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}

POST /ask, token hợp lệ của teacher
HTTP/1.1 200 OK
{"question":"Hello","answer":"...","usage":{"requests_remaining":99,...}}

POST /ask, sau 10 request của student trong 60 giây
HTTP/1.1 429 Too Many Requests
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
retry-after: 43
{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":43}}
```

### Exercise 4.4: Cost guard

`cost_guard.py` kiểm tra budget trước khi gọi LLM và ghi số input/output tokens sau request. Bản demo giới hạn **1 USD/ngày mỗi user**, **10 USD/ngày toàn hệ thống**, cảnh báo khi đạt 80%; user hết budget trả **402**, tổng budget hết trả **503**. Số liệu hiện lưu trong RAM, nên không bền khi restart và không chia sẻ giữa nhiều instance.

Nếu triển khai yêu cầu **10 USD/tháng mỗi user** của Exercise 4.4, lưu chi phí trong Redis với key dạng `budget:<user_id>:<YYYY-MM>`, kiểm tra `current + estimated_cost <= 10` trước request, cập nhật chi phí sau request và đặt TTL đến đầu tháng sau. Phần kiểm tra/cập nhật cần thao tác nguyên tử để các request đồng thời không vượt ngân sách. Rate limit bảo vệ số request theo thời gian; cost guard bảo vệ tổng chi phí token, nên cần cả hai.

**Ghi nhận từ code mẫu:** trường `budget_remaining_usd` trong response `/ask` đang gán `usage.total_cost_usd` (chi phí đã dùng), khác với tên trường. Endpoint `/me/usage` có phép tính ngân sách còn lại đúng hơn.

## Part 5: Scaling & Reliability (CP4)

### Exercise 5.1: Health checks

Bản `develop/app.py` có `GET /health` để báo process còn hoạt động, uptime và tình trạng bộ nhớ; `GET /ready` dùng cờ `_is_ready` để báo có nhận traffic được không (chưa sẵn sàng → 503). Chạy app trên cổng 8005: `/health` → **200**, `status: ok`; `/ready` → **200**, `ready: true`; `POST /ask?question=hello` → **200**. Máy thử chưa cài `psutil`, nên phần kiểm tra bộ nhớ trong `/health` ghi `psutil not installed`.

### Exercise 5.2: Graceful shutdown

`develop/app.py` dùng middleware đếm `_in_flight_requests`. Trong `lifespan`, khi shutdown, app đặt `_is_ready = False` rồi chờ số request đang xử lý về 0, tối đa 30 giây. Uvicorn được cấu hình `timeout_graceful_shutdown=30`. Mục đích là ngừng nhận traffic mới và hoàn tất request hiện tại trước khi thoát. Test SIGTERM đồng thời với request trên Windows chưa được thực hiện; vì vậy chưa khẳng định từ phép thử rằng request đang chạy luôn hoàn tất. Khi dừng một container agent trong stack, log cho thấy các bước `Shutting down` → `Application shutdown complete` → `Finished server process [1]`.

### Exercise 5.3: Stateless design

Bản production lưu session và conversation history trong Redis (`session:<session_id>`, TTL 3600 giây). Mỗi agent đọc/ghi cùng Redis, nên request tiếp theo vẫn tìm được history dù Nginx chuyển sang instance khác. Nếu lưu trong RAM của từng instance, request đến instance khác sẽ không thấy session cũ. Code mẫu có fallback lưu RAM khi không kết nối Redis; chế độ đó không đảm bảo stateless khi scale.

### Exercise 5.4: Load balancing

Chạy `docker compose -p k5-day14-scaling up -d --build --scale agent=3` trong `05-scaling-reliability/production/` sau khi tạo `.env.local` rỗng. Compose khởi động **3 agent**, **1 Redis** và **1 Nginx**; Nginx nhận request trên cổng **8080** rồi proxy đến `agent:8000` trên mạng nội bộ. Năm request trong cùng một session được xử lý bởi ba instance khác nhau: `instance-9bbfdb`, `instance-5a5b81`, `instance-2f1e44`. Sau khi dừng `agent-2` (`instance-9bbfdb`), request tiếp theo vẫn được `instance-5a5b81` xử lý qua Nginx.

### Exercise 5.5: Test stateless

Chạy `python test_stateless.py` với `PYTHONIOENCODING=utf-8` trên PowerShell. Script gửi **5 câu hỏi** trong cùng session; endpoint history trả **10 message** (5 user + 5 assistant) dù các request đi qua ba instance. Sau khi dừng một instance, gửi thêm một câu hỏi bằng cùng `session_id`; history tăng lên **12 message**, response báo `storage: redis`. Điều này chứng minh session được chia sẻ giữa các instance trong phép thử này. Script `test_stateless.py` chỉ gửi 5 câu và đọc history; bước dừng instance được thực hiện riêng. Đã dọn stack bằng `docker compose -p k5-day14-scaling down`.

## Part 6: Final Project (CP5)

Đã hoàn thiện agent trong `06-lab-complete/`: cấu hình qua environment, API key, liveness/readiness, rate limit và cost guard lưu Redis, JSON log, shutdown êm, Docker multi-stage, Compose và `railway.toml`. Chạy `check_production_ready.py` đầy đủ trong container Linux có Docker socket (máy Windows chỉ có distro WSL `docker-desktop`) và nhận kết quả:

```text
📋 Checklist Deliverable
   ✅ Dockerfile (multi-stage, < 500 MB)
   ✅ docker-compose.yml (agent + redis)
   ✅ .dockerignore
   ✅ Health check endpoint (GET /health)
   ✅ Readiness endpoint (GET /ready)
   ✅ API Key authentication
   ✅ Rate limiting
   ✅ Cost guard
   ✅ Config từ environment variables
   ✅ Structured logging
   ✅ Graceful shutdown
   ✅ Public URL ready (Railway / Render config)
Result: 12/12 tiêu chí đạt · 41/41 checks passed
🎉 PASS 12/12 — PRODUCTION READY!
```

Các kết quả chính: image 63 MB theo `docker image inspect`; key thiếu/sai → 401; rate limit 5 request/phút → request kế tiếp 429 và vẫn 429 sau restart; Redis mất kết nối → `/ready` 503 trong khi `/health` 200; budget rất nhỏ → 402; SIGTERM trong lúc đang có request → response 200, process exit 0; `docker stop` → exit 0 trong 0,8 giây. Dockerfile dùng `mirror.gcr.io/library/python:3.11-slim` vì Docker Hub trả HTTP 429 khi build trên máy này.

## Part 7: Benchmark local (CP7)

Đã chạy agent v1 trên cổng 8001, v2 trên cổng 8002. `load_test.py` gửi 300 request cho từng mức concurrency 1, 10, 50, 100 và từng mức `MOCK_SLOW_RATE` 0, 0.03, 0.1 (concurrency 20). JSON kết quả nằm tại `06-lab-complete/benchmark-observability/develop/reports/`; các bảng số liệu và nhận xét ở `BENCHMARK_REPORT.md`. Concurrency 50 và 100 trượt SLO p95 500 ms; mức slow rate 0.1 cũng trượt SLO p95.

Eval trên 23 câu: v1 đúng 1/23, v2 ban đầu đúng 21/23, v2 sau khi thêm hai entry cho JWT và Kubernetes HPA đúng 23/23. Lệnh cuối với `--min-accuracy 1.0 --baseline reports/eval_v2.json` trả **exit code 0, Quality gate PASS**, không regression. Chạy v1 với baseline v2 trả **exit code 1, Quality gate FAIL**, có 20 regression.

**Cách chặn merge trong CI:** khởi động agent candidate, chạy `python eval/run_eval.py --url http://localhost:8000 --baseline reports/eval_v2_fixed.json --min-accuracy 1.0 --out reports/eval_candidate.json` như một step bắt buộc trong GitHub Actions. Không dùng `continue-on-error`; khi script trả exit code 1 do accuracy thấp hoặc regression, job fail và branch protection không cho merge. Số liệu eval không dùng để kết luận p95 vì chỉ có 23 request.

## CP8 — Observability stack và benchmark scaling

Đã chạy Docker Compose project `k5-day14-observability` với agent, Nginx, Prometheus, Grafana, Jaeger và Locust. Ảnh bằng chứng gồm `screenshots/grafana-dashboard.png`, `screenshots/jaeger-slow-trace.png`, `screenshots/jaeger-error-trace.png`, `screenshots/prometheus-alert-firing.png`. `promtool test rules` trả `SUCCESS`; snapshot Prometheus API ghi nhận đồng thời bốn alert firing.

Locust headless 100 users, ramp 10/s, 2 phút: 1 instance đạt 54.21 RPS, p95 1700 ms; 3 instances đạt 59.29 RPS, p95 370 ms. File CSV ở `06-lab-complete/benchmark-observability/production/benchmark-results/`. Dùng `docker compose run --no-deps` để Compose giữ đúng số instance của phép đo.

Trace chậm `03db16d361d8181eaf3f2a98a9e9c49d` có tổng 2308.5 ms, span LLM 1705.4 ms. Trace lỗi `c2d75201c75e1a2b2683d2816b18c27a` có event `exception` và log `llm_error` cùng `trace_id`. `MOCK_ERROR_RATE=0.2` làm alert error firing sau khoảng 84 giây; `MOCK_SLOW_RATE=0.3` làm alert p95 firing sau khoảng 139 giây; pause một agent làm alert down firing sau khoảng 35–42 giây. Cost alert đã firing từ tải baseline, nên không thể gán thời gian kích hoạt riêng cho thay đổi model. Đổi sang v1 cho eval 1/23 câu đúng và 22 regression, quality gate FAIL; cần eval theo lịch để phát hiện lỗi chất lượng mà metric HTTP không thấy. Chi tiết và dòng log JSON ở [BENCHMARK_REPORT.md](BENCHMARK_REPORT.md).

## AI log

Bài lab có sử dụng ChatGPT/Codex để giải thích yêu cầu, đọc lỗi và hỗ trợ viết code, tài liệu. Các số đo và trạng thái được ghi theo kết quả lệnh, JSON/CSV và ảnh trong repo.

| Nội dung đã hỏi AI | AI hỗ trợ | Cách kiểm tra kết quả |
|---|---|---|
| CP1: tóm tắt repo, bắt đầu lab, sửa lệnh `curl` trên Windows và điền Exercise 1.1 | Giải thích cấu trúc, cú pháp PowerShell và sáu anti-pattern | Gọi app local, `/ask` trả 200; so với code develop/production. |
| CP2: triển khai Docker và kiểm tra Exercise 2.3 | Hỗ trợ lệnh build, Compose và sửa tài liệu bị trùng mục | Đo image develop 1,67 GB, production 240 MB; kiểm tra `/health` và `/ask` qua Nginx. |
| CP3: tiếp tục bảo mật API | Giải thích API key, JWT, rate limit và cost guard; hỗ trợ ghi đáp án | Kiểm tra thực tế 401, 200, 429 và nội dung response như Part 4. |
| CP4: tiếp tục scaling và reliability | Hỗ trợ chạy ba instance và phân tích session dùng chung Redis | Chạy `test_stateless.py`, kiểm tra history 10 rồi 12 message sau khi dừng một instance. |
| CP5: hoàn thiện agent production | Hỗ trợ sửa code/config, đọc lỗi và chạy checker | Lần chạy Linux trước đó báo PASS 12/12, 41/41 checks; output tóm tắt ở Part 6. |
| CP6: hướng dẫn deploy Render và kiểm tra 401/200 | Hướng dẫn cấu hình Web Service, Redis và lệnh PowerShell | URL Render trả `/health`, `/ready` 200; `/ask` không key 401, có key 200; lưu ảnh deploy và request. |
| CP7: benchmark tải và eval | Hỗ trợ chạy công cụ, đọc JSON, sửa hai mục knowledge base và viết nhận xét | Đối chiếu `reports/load_v2_*.json`, `reports/eval_*.json`; quality gate v2 sau sửa đạt 23/23. |
| CP8: Docker Compose observability, Locust, Jaeger, Prometheus | Hỗ trợ sửa Dockerfile của agent quan sát, thêm `--no-deps` để phép đo giữ đúng số replicas, chạy thí nghiệm và điền báo cáo | Đối chiếu `benchmark-results/scale_*.csv`, trace JSON, snapshot alert và bốn ảnh CP8; `promtool test rules` báo SUCCESS. |

Không dùng AI để dựng hay sửa nội dung ảnh bằng chứng hoặc tự đặt số đo; ảnh được chụp từ giao diện thực. Người nộp cần đọc lại phần phân tích và file nộp để có thể giải thích khi Lab Coach hỏi.
