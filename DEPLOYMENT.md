# Thông Tin Deploy — Checkpoint 5

> Service agent và Redis đã được deploy thành công trên cloud (Railway) với domain HTTPS công khai. Repo không chứa giá trị secret.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đàm Quang Sơn |
| Mã học viên | 2A202602868 |
| Repo | https://github.com/KuanqXol/K4-L3A-DAY12-DamQuangSon-2A202602868-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-l3a-day12-damquangson-2a202602868-cloudservic-production.up.railway.app |
| Platform | Railway |
| Trạng thái | Đang hoạt động (Online, HTTPS) |
| Ngày kiểm tra | 2026-09-28 |

## Cấu Hình Đã Dùng

Chỉ liệt kê tên biến và nguồn, không ghi giá trị secret:

| Biến | Trạng thái | Nguồn |
|------|------------|-------|
| `PORT` | ✅ | Do Railway tự cấp lúc khởi động container |
| `AGENT_API_KEY` | ✅ | Biến môi trường Railway (Secret Variable) |
| `REDIS_URL` | ✅ | Kết nối qua Redis service nội bộ trên Railway (`${{ Redis.REDIS_URL }}`) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | Biến môi trường Railway, giá trị 10 |
| `MONTHLY_BUDGET_USD` | ✅ | Biến môi trường Railway, giá trị 10.0 |
| `LOG_LEVEL` | ✅ | Biến môi trường Railway, mức INFO |

## Kiến Trúc Cloud

- **Agent Service**: Build từ Dockerfile multi-stage, chạy trên Railway với domain HTTPS công khai.
- **Redis Service**: Database Redis provisioned trực tiếp trên Railway, kết nối qua mạng nội bộ `redis.railway.internal:6379`.
- Lưu trữ session, history, sliding window rate limiter và monthly cost guard trên Redis tập trung.

## Lệnh Kiểm Tra

```bash
# 1. Kiểm tra liveness
curl -i https://k4-l3a-day12-damquangson-2a202602868-cloudservic-production.up.railway.app/health

# 2. Kiểm tra readiness & kết nối Redis
curl -i https://k4-l3a-day12-damquangson-2a202602868-cloudservic-production.up.railway.app/ready

# 3. Kiểm tra bảo vệ endpoint /ask khi không có API key (phải trả 401)
curl -i -X POST https://k4-l3a-day12-damquangson-2a202602868-cloudservic-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Kiểm tra endpoint /ask với API key hợp lệ (trả 200)
curl -i -X POST https://k4-l3a-day12-damquangson-2a202602868-cloudservic-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: deployment-evidence" \
  -d '{"question":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

```text
GET /health
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

GET /ready
HTTP/1.1 200 OK
{"status":"ready","redis":true}

POST /ask không có API key
HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

POST /ask có API key
HTTP/1.1 200 OK
{"answer":"Ngắn gọn: Deploy là quá trình đưa mã nguồn ứng dụng từ môi trường phát triển lên môi trường máy chủ (server/cloud) để người dùng cuối có thể truy cập và sử dụng dịch vụ.","user_id":"deployment-evidence","history_length":2,"cost_usd":0.0003,"tokens":{"in":5,"out":42}}
```

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png` — Dashboard dịch vụ trên Railway (gồm service agent và database Redis).
- `screenshots/health.png` — Kết quả gọi thực tế `/health` và `/ready` qua domain HTTPS public.
