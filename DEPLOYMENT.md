# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục         | Nội dung                                                             |
| ----------- | -------------------------------------------------------------------- |
| Họ và tên   | Nguyễn Minh Đức                                                      |
| Mã học viên | 2A202601438                                                          |
| Repo        | https://github.com/MinhDuc-IT/K4-Day12-2A202601438-NguyenMinhDuc.git |

## Service

| Mục         | Nội dung                                                             |
| ----------- | -------------------------------------------------------------------- |
| Public URL  | https://k4-day12-2a202601438-nguyenminhduc-production.up.railway.app |
| Platform    | Railway                                                              |
| Ngày deploy | 10/08/2026                                                           |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến                | Đã set | Ghi chú                                                |
| ------------------- | ------ | ------------------------------------------------------ |
| `PORT`              | ✅     | Railway tự gán khi container khởi động                 |
| `API_TOKEN`         | ✅     | Đặt thủ công trong Railway dashboard → Variables       |
| `REDIS_URL`         | ✅     | Railway Redis add-on, reference `${{Redis.REDIS_URL}}` |
| `BUCKET_CAPACITY`   | ✅     | 10                                                     |
| `REFILL_PER_MINUTE` | ✅     | 10                                                     |
| `DAILY_BUDGET_USD`  | ✅     | 1.0                                                    |
| `LOG_LEVEL`         | ✅     | INFO                                                   |

## Lệnh Kiểm Tra

```bash
URL=https://k4-day12-2a202601438-nguyenminhduc-production.up.railway.app

# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i $URL/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i $URL/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST $URL/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST $URL/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST $URL/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Kiểm tra lúc 10/08/2026 (sau khi sửa startCommand expand PORT):

```
# 1. GET /healthz
HTTP/1.1 200 OK
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

# 2. GET /readyz
HTTP/1.1 200 OK
{"status":"ready","redis":true}

# 3. POST /chat (không token)
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên Railway (Deploy Success)
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không áp dụng — đã deploy thành công trên Railway.
