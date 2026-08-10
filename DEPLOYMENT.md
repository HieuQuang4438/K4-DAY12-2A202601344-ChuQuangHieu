# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Chu Quang Hiếu |
| Mã học viên | 2A202601344 |
| Repo | https://github.com/HieuQuang4438/K4-DAY12-2A202601344-ChuQuangHieu |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-chat-xsmh.onrender.com/ |
| Platform | Render |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard Render (`sync: false`), không nằm trong repo |
| `REDIS_URL` | ✅ | Redis instance trên Render, nối qua `fromService` trong `render.yaml` |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Chạy ngày 10/08/2026, `<URL>` = `https://day12-chat-xsmh.onrender.com`.

```
# 1. GET /healthz
HTTP/1.1 200 OK
Content-Type: application/json
x-render-origin-server: uvicorn

{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

# 2. GET /readyz
HTTP/1.1 200 OK
Content-Type: application/json
x-render-origin-server: uvicorn

{"status":"ready","redis":true}

# 3. POST /chat — không gửi token
HTTP/1.1 401 Unauthorized
Content-Type: application/json
www-authenticate: Bearer

{"detail":"invalid or missing bearer token"}

# 4. POST /chat — token hợp lệ
HTTP/1.1 200 OK
Content-Type: application/json

{"reply":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến
môi trường, health check để orchestrator biết trạng thái, và giới hạn tài
nguyên.","client_id":"sv-test","turns_before":0,"usd_cost":2.265e-05,
"usage":{"prompt":3,"completion":37}}

# 5. Rate limit — 15 request liên tiếp, X-Client-Id: sv-ratelimit
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

Nhận xét: đúng 10 request đầu đi qua, bằng `BUCKET_CAPACITY=10`. 5 request sau
bị token bucket chặn 429 vì xô đã cạn, `REFILL_PER_MINUTE=10` chưa kịp nhỏ thêm
token trong vài giây đó.

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

Không dùng phương án dự phòng — service chạy thật trên Render, `LOCAL_FALLBACK=false`.
