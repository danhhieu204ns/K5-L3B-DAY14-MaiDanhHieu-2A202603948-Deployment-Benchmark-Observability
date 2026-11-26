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
