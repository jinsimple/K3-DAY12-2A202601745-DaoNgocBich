# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | DaoNgocBich |
| Mã học viên | 2A202601745 |
| Repo | https://github.com/jinsimple/K3-Day12-Cloud-Services-And-Deployment2A202601745-DaoNgocBich |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-3ea9.up.railway.app |
| Platform | Railway (service `agent` build từ Dockerfile + Redis add-on trong cùng project) |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán, app đọc qua `${PORT:-8000}` trong CMD |
| `AGENT_API_KEY` | ✅ | set qua `railway variable set` trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | tham chiếu tới Redis add-on cùng project bằng `${{Redis.REDIS_URL}}` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

```
# 1. Liveness — mong đợi 200 {"status":"ok"}
$ curl -i https://agent-production-3ea9.up.railway.app/health
HTTP/2 200
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness — mong đợi 200 {"status":"ready"}
$ curl -i https://agent-production-3ea9.up.railway.app/ready
HTTP/2 200
{"status":"ready","redis":true}

# 3. Không có API key — mong đợi 401
$ curl -i -X POST https://agent-production-3ea9.up.railway.app/ask -d '{"question":"Hello"}'
HTTP/2 401
{"detail":"invalid or missing API key"}

# 4. Có API key — mong đợi 200 kèm câu trả lời
$ curl -i -X POST https://agent-production-3ea9.up.railway.app/ask -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Deploy là gì?"}'
HTTP/2 200
{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa
môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.",
"user_id":"sv-test","history_length":4,"cost_usd":3.915e-05,"tokens":{"in":81,"out":45}}

# 5. Rate limit — gọi 15 lần liên tiếp, mong đợi các lần cuối trả 429
$ for i in $(seq 1 15); do curl ... ; done
200 200 200 200 200 200 200 200 200 429 ...
```

Ghi chú trung thực: khi gọi dồn dập nhiều lần từ máy cá nhân qua Internet, một
vài request thỉnh thoảng bị timeout ở tầng edge của Railway (free tier) chứ
không phải app trả lỗi — log `railway logs --service agent` trong lúc đó không
ghi nhận exception nào phía app, và không có request nào tới được container.
Việc rate limit (429) hoạt động đúng đã được xác nhận nhiều lần cả qua curl
trực tiếp lẫn qua log server thật.

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

Đã deploy thật lên Railway — không dùng phương án dự phòng.
