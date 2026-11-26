# Deployment Information

## Public URL

https://k5-l3b-day14-maidanhhieu-2a202603948.onrender.com

## Platform

Render Web Service, Docker build từ `06-lab-complete/`, branch `main`. Render dashboard báo deploy thành công và Live ngày 26/11/2026, source commit `51017d1`.

## Public endpoint checks

| Request | Kết quả đo từ URL public |
|---|---:|
| `GET /health` | 200 |
| `GET /ready` | 200 |
| `POST /ask` không có `X-API-Key` | 401 |
| `POST /ask` có `X-API-Key` đúng | 200; JSON có `question`, `answer`, `cost_usd` |

`GET /` trả 404 vì app không khai báo route gốc. `/ready` trả 200 cho thấy app kết nối được Redis tại thời điểm kiểm tra.

## Test authenticated request (PowerShell)

Trước tiên đặt một `AGENT_API_KEY` mới trong Render Dashboard nếu key cũ đã lộ. Chờ service redeploy xong. Nhập key qua prompt ẩn dưới đây; **không đặt key vào chuỗi lời nhắc hoặc dán vào báo cáo**:

```powershell
$base = 'https://k5-l3b-day14-maidanhhieu-2a202603948.onrender.com'
$secureKey = Read-Host 'Nhap AGENT_API_KEY moi' -AsSecureString
$ptr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureKey)
try { $key = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($ptr) }
finally { [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($ptr) }
$response = Invoke-WebRequest -UseBasicParsing -Method Post -Uri "$base/ask" -Headers @{'X-API-Key'=$key} -ContentType 'application/json' -Body '{"question":"Hello"}'
$response.StatusCode
$response.Content
Remove-Variable key,secureKey,response
```

Kỳ vọng: HTTP 200 và JSON có `question`, `answer`, `cost_usd`. Nếu vẫn 401, kiểm tra trên Render rằng biến `AGENT_API_KEY` thuộc đúng Web Service và bản deploy mới đã Live.

## Environment variables

Agent cần `AGENT_API_KEY`, `REDIS_URL`, `ENVIRONMENT`, `RATE_LIMIT_PER_MINUTE`, `DAILY_BUDGET_USD`; giá trị thực được đặt trên Render và không lưu trong repo. Không cần OpenAI API key vì lab dùng mock LLM.

## Screenshots

- [Render Key Value đang Available](screenshots/dashboard.png)
- [POST /ask có key trả 200](screenshots/test.png)

Cần thêm ảnh **Web Service báo Deploy succeeded / Live** (lưu `screenshots/web-live.png`) để bằng chứng deployment rõ ràng. Ảnh không được lộ API key.
