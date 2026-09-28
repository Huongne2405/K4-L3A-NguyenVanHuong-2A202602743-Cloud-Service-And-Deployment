# Thông Tin Deploy — Checkpoint 5

> Service được deploy công khai trên Railway. Tài liệu chỉ ghi tên biến môi
> trường và nguồn cấu hình, không chứa giá trị API key.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Văn Hưởng |
| Mã học viên | 2A202602743 |
| Repo | https://github.com/Huongne2405/K4-L3A-NguyenVanHuong-2A202602743-Cloud-Service-And-Deployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-00a4.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | tham chiếu tới Redis service trong cùng Railway project |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://agent-production-00a4.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://agent-production-00a4.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://agent-production-00a4.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://agent-production-00a4.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://agent-production-00a4.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

```text
GET /health
HTTP 200
{"status":"ok","service":"day12-agent","version":"1.0.0"}

GET /ready
HTTP 200
{"status":"ready","redis":true}

POST /ask (không có API key)
HTTP 401
{"detail":"invalid or missing API key"}

POST /ask (có API key hợp lệ)
HTTP 200
{"user_id":"cp5-verification","history_length":0,"answer":"..."}

Rate limit (15 request, cùng một user)
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
