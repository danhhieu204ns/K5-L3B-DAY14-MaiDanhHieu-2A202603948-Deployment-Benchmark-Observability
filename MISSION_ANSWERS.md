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
