# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Minh Đức  Mã học viên: 2A202601438

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu quên set `API_TOKEN` trong Variables thì app crash ngay lúc start với `ValidationError: api_token Field required`. Mình thấy lỗi trên dashboard/logs và sửa trước khi có traffic thật. Nếu mặc định là `"changeme"`, service vẫn “chạy xanh”, healthcheck pass, nhưng ai biết token mặc định cũng gọi được `/chat` — mình chỉ phát hiện khi hóa đơn LLM tăng hoặc khi có người lạ spam API.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật khi chat thành công:
> `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:11:17.358053+00:00", "client_id": "sv-test", "prompt_tokens": 3, "completion_tokens": 40, "usd_cost": 0.000024}`
>
> (1) Lọc/đếm theo trường: ví dụ gom theo `client_id` hoặc tổng `usd_cost` trong ngày trên Cloud Logging. (2) Cảnh báo theo `severity` hoặc ngưỡng chi phí — `print("đã trả lời xong")` không có cấu trúc nên máy không query/alert được.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1800 MB (base `python:3.11` đầy đủ + deps + build tools nằm chung image) |
| Multi-stage | ~250–350 MB (`python:3.11-slim`, chỉ copy `/install` từ builder; CP2 test yêu cầu < 400MB và đã pass) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch chủ yếu là phần **không cần lúc chạy**: compiler/`build-essential`, cache pip, toolchain của stage builder, và base image đầy đủ nặng hơn `slim`. Multi-stage vứt stage builder sau khi `pip install --prefix=/install`, runtime chỉ giữ thư viện đã cài + source `app/`/`utils/`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile hiện tại: `COPY requirements.txt` → `pip install` → rồi mới `COPY app` / `COPY utils`. Sửa một ký tự trong `main.py` thì các layer đến hết `pip install` vẫn **cache**; chỉ layer copy code (+ các layer sau) build lại.
>
> Nếu `COPY . .` đứng trước `pip install`, mọi thay đổi source làm invalidate layer copy → `pip install` chạy lại từ đầu dù `requirements.txt` không đổi → build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng RCE trong app → process trong container chạy **root** → kẻ tấn công có quyền root *trong* container → nếu có lỗ hổng escape/container misconfig (mount Docker socket, volume nhạy cảm, kernel bug…) dễ leo lên quyền cao trên host.
>
> Lệnh `USER appuser` cắt chuỗi ngay từ đầu: process app chỉ là user thường (uid 10001). Dù bị RCE, họ không mặc định là root trong container, giảm bề mặt leo thang sang host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là quy ước HTTP/RFC 6750: client biết phải xác thực bằng Bearer token, không phải Basic hay kiểu khác.
>
> Cùng một message (`invalid or missing bearer token`) để không lộ thông tin cho kẻ dò: phân biệt “sai scheme” vs “sai token” giúp họ biết mình đã gần đúng format hay gần đúng giá trị token.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Im lặng 10 phút: xô đã đầy tối đa `capacity=10` (không tích quá sức chứa). Gửi liên tiếp được **10 request**, request thứ 11 → 429.
>
> Bỏ `min(capacity, ...)`: sau 10 phút có thể tích ~100 token (10/phút × 10 phút). Gửi liên tiếp được khoảng **100 request** trước 429 — burst quá lớn, đúng cái capacity muốn chặn.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> **$30/tháng:** từ 2h sáng có thể đốt gần hết ngân sách tháng trong vài giờ → thiệt hại tối đa ~$30; phải đợi sang tháng (hoặc reset tay) mới dùng lại.
>
> **$1/ngày:** thiệt hại tối đa trong ngày sự cố ~$1; sang ngày UTC mới key `spend:...:YYYY-MM-DD` khác → ngân sách tự về 0 đã tiêu, service tự “hồi” mà không cần can thiệp. Cùng mức ~$30/tháng nhưng giới hạn thiệt hại một sự cố xuống khoảng 1/30.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1) Redis mất kết nối 30s. 2) Cả 3 instance gọi probe “gộp” đều fail vì không ping được Redis. 3) Orchestrator coi cả 3 **unhealthy** và **restart** đồng loạt. 4) Trong lúc restart, không còn instance phục vụ → outage toàn cụm. 5) Redis sống lại nhưng traffic vẫn gián đoạn vì đang restart.
>
> Tách đúng: `/healthz` vẫn 200 (process sống) → không restart; `/readyz` 503 → LB chỉ ngừng đẩy request vào instance đó. Redis về lại thì ready lại, không kéo sập cả cụm.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi:** Railway deploy FAILED ở bước **Network > Healthcheck** (Build/Deploy Success). Public URL trả `404 Application not found` với `x-railway-fallback: true`.
>
> **Cách tìm:** Xem dashboard Deployments + mô phỏng startCommand local: chạy `uvicorn ... --port $PORT` mà `$PORT` không qua shell → `Invalid value for '--port': '$PORT' is not a valid integer` / `--port requires an argument`. `startCommand` trong `railway.toml` ghi đè `CMD` Dockerfile nên fallback `${PORT:-8000}` trong Dockerfile không được dùng.
>
> **Cách sửa:** đổi `startCommand` thành `sh -c 'uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'` và tăng `healthcheckTimeout`. Redeploy xong `/healthz` 200, `/readyz` 200, `/chat` không token → 401.
