# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: DaoNgocBich  Mã học viên: 2A202601745

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Giả sử tôi deploy lên Railway và quên set biến `AGENT_API_KEY` trong dashboard.
> Nếu `Settings` có default `"changeme"`, app vẫn khởi động bình thường, `/health`
> vẫn xanh, và bất kỳ ai đọc source code trên GitHub (repo public theo yêu cầu
> nộp bài) cũng biết được giá trị mặc định đó — họ chỉ cần gửi
> `X-API-Key: changeme` là gọi được `/ask` miễn phí bằng ngân sách của tôi. Tôi sẽ
> không biết có chuyện gì bất thường cho tới khi thấy hóa đơn cloud tăng vọt hoặc
> cost guard báo hết ngân sách tháng — lúc đó thiệt hại đã xảy ra rồi.
>
> Vì `agent_api_key: str` không có default, `Settings()` ném `ValidationError`
> ngay trong `lifespan`, trước khi `yield` — container crash ngay lúc khởi động,
> Railway hiển thị trạng thái đỏ trên dashboard deploy. Tôi biết ngay lập tức,
> đúng lúc tôi đang theo dõi màn hình deploy, thay vì phát hiện qua một sự cố âm
> thầm vài ngày sau.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật thu được khi gọi `/ask`:
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:17:17.667495+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
> ```
>
> Hai việc làm được mà `print()` không làm được:
> 1. Lọc/tổng hợp theo trường: ví dụ `jq 'select(.user_id=="sv01") | .cost_usd' `
>    để tính tổng chi phí của một user trong ngày — `print()` là chuỗi tự do,
>    không có trường để lọc.
> 2. Cảnh báo tự động: Datadog/CloudWatch có thể tạo alert "nếu `level=error`
>    xuất hiện > 10 lần/phút" vì chúng parse được JSON theo khóa; với
>    `print("đã trả lời xong")` không có cách nào phân biệt log lỗi và log bình
>    thường bằng máy.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1730 MB (1.73 GB) |
| Multi-stage | 296 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch ~1.43GB, chủ yếu đến từ 3 nguồn: (1) **base image**: `python:3.11`
> (bản đầy đủ) đã nặng hơn `python:3.11-slim` cả trăm MB ngay từ đầu vì nó cài
> kèm nhiều gói hệ thống, header dev, mã nguồn Python đầy đủ mà stage runtime
> không cần tới. (2) **build tool/compiler**: một số dependency trong
> `requirements.txt` cần biên dịch native (C extension), nên môi trường build
> cần `gcc`/`build-essential`; bản 1-stage giữ nguyên các gói này trong layer
> cuối cùng vì `pip install` chạy ngay trên stage duy nhất, còn multi-stage cô
> lập việc cài đặt ở stage `builder` và chỉ `COPY --from=builder /install
> /usr/local` — mang đúng site-packages đã cài xong, bỏ lại toàn bộ compiler.
> (3) **cache & source thừa**: bản 1-stage dùng `COPY . .` nên mang theo cả
> `.git`-adjacent files, test, cache pip (`~/.cache/pip` không dọn vì thiếu
> `--no-cache-dir`) vào chung một layer với source code — layer đó không thể
> "bỏ bớt" được nữa vì đã nằm chung stage cuối cùng của image.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Đã thực nghiệm thật: build multi-stage một lần, thêm một dòng trắng vào cuối
> `app/main.py`, build lại — output thật:
> ```
> #6 [builder 1/4] FROM python:3.11-slim               CACHED
> #7 [builder 4/4] RUN pip install --no-cache-dir ...   CACHED
> #9 [builder 3/4] COPY requirements.txt .              CACHED
> #10 [stage-1 3/6] COPY --from=builder /install ...    CACHED
> #11 [stage-1 4/6] COPY app/ app/                      (chạy lại — không CACHED)
> #12 [stage-1 5/6] COPY utils/ utils/                  (chạy lại)
> #13 [stage-1 6/6] RUN useradd ...                     (chạy lại)
> ```
> Toàn bộ layer ở stage `builder` (cài dependency) và layer `COPY --from=builder`
> ở stage runtime đều được dùng lại từ cache — vì input của chúng
> (`requirements.txt`) không đổi. Chỉ từ `COPY app/ app/` trở đi (layer đầu
> tiên phụ thuộc trực tiếp vào file vừa sửa) mới phải chạy lại, và mọi layer
> **sau** nó trong cùng stage cũng phải chạy lại theo (Docker cache theo thứ
> tự tuyến tính — một layer đổi thì mọi layer phía sau nó mất cache dù bản
> thân chúng không đổi).
>
> Nếu đặt `COPY . .` **trước** `RUN pip install` (ngược thứ tự): bất kỳ thay
> đổi nào trong code — kể cả sửa một dấu phẩy trong comment — cũng làm layer
> `COPY . .` đổi hash, kéo theo `RUN pip install` phía sau luôn phải chạy lại
> từ đầu dù `requirements.txt` không hề đổi. Với dependency nặng (ví dụ có
> compile native), một vòng code–build–test có thể chậm thêm hàng chục giây
> tới vài phút một cách không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python của tôi có lỗ hổng, ví dụ một endpoint nào đó
> deserialize input người dùng không kiểm soát chặt, cho phép kẻ tấn công thực
> thi lệnh tùy ý (remote code execution) bên trong container. (2) Vì process
> `uvicorn` đang chạy bằng `root` (UID 0) — mặc định của container nếu không
> khai báo `USER` — nên lệnh tùy ý đó cũng chạy với quyền root *bên trong*
> container: đọc/ghi được mọi file trong container, cài package, sửa
> `/etc/passwd`... (3) Nếu container engine có misconfiguration (chạy
> `--privileged`, mount `/var/run/docker.sock`, hoặc gặp lỗ hổng kernel
> container-escape như các CVE liên quan tới `runc`), quyền root *bên trong*
> container có thể leo thang thành quyền root *trên host* — kẻ tấn công thoát
> khỏi container, chạm được vào các container khác hoặc chính máy chủ.
>
> Lệnh `USER appuser` (thêm sau khi tạo user không có quyền admin trong
> Dockerfile) cắt đứt chuỗi này ở bước (2): dù lỗ hổng ở bước (1) vẫn tồn tại và
> kẻ tấn công vẫn thực thi được lệnh, lệnh đó chỉ chạy với quyền của một user
> thường — không đọc/ghi được file hệ thống, không cài được package cần quyền
> root, và ngay cả khi container-escape xảy ra ở bước (3), UID leo ra ngoài vẫn
> là UID thường chứ không phải root, giảm hẳn mức độ nghiêm trọng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt được: người dùng gửi đúng 10
> request lúc 10:00:59 (giây cuối cùng của cửa sổ phút "10:00") — bộ đếm của
> phút đó ghi nhận 10/10, vẫn hợp lệ. Ngay khi đồng hồ sang 10:01:00, bộ đếm
> reset về 0 vì đây là một "phút đồng hồ" mới, nên người dùng gửi tiếp 10
> request nữa lúc 10:01:01 — lại đúng 10/10 của phút "10:01", vẫn hợp lệ theo
> luật đếm-theo-phút. Kết quả: 20 request lọt qua trong khoảng 2 giây thực tế
> (từ 10:00:59 đến 10:01:01), dù hạn mức công bố là "10/phút". Sliding window
> không có kẽ hở này vì nó luôn nhìn lại đúng 60 giây gần nhất tính từ *thời
> điểm request tới* (`zremrangebyscore(key, 0, now-60)`), không có ranh giới cố
> định để "né" bằng cách canh giờ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **tần suất** (số request trong một khoảng thời gian),
> không quan tâm mỗi request nặng hay nhẹ. Cost guard giới hạn **tiền thực tế
> đã tiêu** trong tháng, không quan tâm request đó có đến nhanh hay chậm. Hai
> trục độc lập nhau nên không thể thay thế nhau.
>
> - Rate limit cho qua nhưng cost guard chặn: hạn mức 10 request/phút, user chỉ
>   gửi 3 request trong phút này (rate limit thoải mái cho qua) — nhưng mỗi
>   request là một câu hỏi cực dài, 50.000 token, tổng chi phí 3 request đó đã
>   vượt ngân sách tháng còn lại. Cost guard phải chặn ở request thứ 3 dù rate
>   limit không hề phàn nàn.
> - Cost guard cho qua nhưng rate limit chặn: user gửi 20 request/giây, mỗi
>   request chỉ hỏi "hi" (vài chục token, gần như miễn phí) — tổng chi phí rất
>   nhỏ so với ngân sách tháng nên cost guard không có lý do chặn. Nhưng tần
>   suất 20 request/giây vượt xa hạn mức 10/phút, khiến rate limit chặn ngay từ
>   request thứ 11 trong phút đó — bảo vệ hệ thống khỏi bị spam/DoS dù chi phí
>   mỗi request rẻ.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện: (1) Redis mất kết nối. (2) Endpoint gộp — vốn đóng cả vai trò
> liveness lẫn readiness — gọi `store.ping()`, thất bại, trả 503 ở cả 3 container
> cùng lúc (vì cả 3 đều dùng chung một Redis). (3) Orchestrator (Docker/K8s/
> Railway) đang dùng chính endpoint này làm **liveness probe**, hiểu 503 là
> "process chết, cần restart" — nó restart cả 3 container gần như đồng thời.
> (4) Container mới khởi động lại vẫn gọi cùng endpoint đó để kiểm tra, Redis
> vẫn chưa sống lại trong 30 giây đó, container mới lại nhận 503 → bị coi là
> chết → restart tiếp — vòng lặp restart liên tục (crash loop). (5) Trong suốt
> 30 giây đó, **không có container nào ở trạng thái ổn định để nhận request** —
> toàn bộ traffic bị gián đoạn hoàn toàn, kể cả những request không hề cần đến
> Redis, thay vì chỉ tạm dừng đẩy traffic mới. (6) Khi Redis sống lại, các
> container vẫn đang trong chu kỳ restart phải khởi động lại hoàn chỉnh (load
> lại process, mở kết nối...) mới phục vụ trở lại được — độ trễ khôi phục dài
> hơn hẳn so với việc tách riêng `/ready`, nơi container chỉ đơn giản bị rút
> tạm khỏi load balancer rồi được đưa lại vào ngay khi Redis ping thành công,
> không phải restart gì cả.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Đã thực nghiệm thật với `docker compose up -d --scale agent=3` (3 container
> `agent` phía sau Nginx round-robin), gọi `/ask` liên tiếp với cùng
> `X-User-Id: sv01`, kết quả `history_length` trong response:
> ```
> 0 → 2 → 4 → 6 → (lần thứ 5 bị chặn 429 rate limit exceeded — cộng dồn từ
>                  các lần gọi test trước đó trong cùng phút, đúng như CP3)
> ```
> Tăng đều 2 mỗi lần (1 lượt `user` + 1 lượt `assistant` được `store.append`
> sau mỗi câu trả lời) dù mỗi request có thể rơi vào một trong 3 container
> `agent` khác nhau (Nginx round-robin theo `nginx/nginx.conf`) — vì cả 3 đều
> đọc/ghi chung một Redis (`REDIS_URL=redis://redis:6379/0`), không container
> nào giữ state riêng.
>
> Nếu lịch sử được lưu trong `dict` Python trong process (thay vì Redis):
> `history_length` sẽ **nhảy lung tung, không tăng đều** — mỗi container có
> một `dict` riêng trong bộ nhớ của chính nó; request nào rơi vào container A
> thấy lịch sử của container A, request tiếp theo rơi vào container B (không
> biết gì về những gì vừa xảy ra ở A) sẽ thấy `history_length` nhỏ hơn hoặc
> bằng 0 dù đã hỏi nhiều lần trước đó — service không stateless, không scale
> ngang đúng nghĩa được.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **CHƯA ĐIỀN — cần deploy thật lên Railway/Render (CP5), việc mà chỉ bạn làm
> được vì cần tài khoản cá nhân của bạn.**
> Đây là câu bắt buộc phải là trải nghiệm thật của riêng bạn — không nên nhờ
> AI viết hộ, vì Lab Coach có thể hỏi trực tiếp "lỗi này bạn debug thế nào".
> Khi deploy, các lỗi thường gặp (ghi lại đúng cái bạn thật sự gặp):
> - App không bind đúng cổng cloud gán (`$PORT` động, không phải cố định 8000)
>   → sửa bằng cách đọc `settings.port` từ env thay vì hardcode.
> - `REDIS_URL` trỏ sai (thiếu, hoặc dùng `localhost` thay vì hostname nội bộ
>   của Railway/Render) → sửa bằng cách set đúng biến môi trường trên dashboard.
> - Health check timeout vì `/health` bị định nghĩa nhầm phụ thuộc vào Redis
>   (xem Câu 8) → sửa bằng cách tách `/health` khỏi mọi dependency.
> Điền đúng lỗi thật bạn gặp, thông báo lỗi nguyên văn, và các bước debug thật
> (đọc log trên dashboard, `docker logs`, thử curl thủ công...).
