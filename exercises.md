# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder "Câu trả lời của bạn" bên dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Minh Điềm  Mã học viên: 2A202602858

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: deploy lên Render, tôi quên nhập `AGENT_API_KEY` trong dashboard.
> Với fail fast, `Settings()` ném `ValidationError: agent_api_key Field required`
> ngay lúc khởi động, deploy báo fail, health check không lên → tôi thấy ngay trong log và > sửa trước khi có người dùng. Nếu mặc định là `"changeme"`, app vẫn khởi động, health 
> check xanh, trông như deploy thành công; nhưng `/ask` trên URL công khai được bảo vệ bằng
> một khóa mà ai cũng đoán được → người lạ truy cập được LLM

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật khi gọi `/ask` ở máy (docker compose):
>
> ```
> {"event":"ask_completed","level":"info","timestamp":"2026-09-29T05:12:40.629401+00:00","user_id":"sv01","tokens_in":49,"tokens_out":47,"cost_usd":3.555e-05}
> ```
>
> 1. **Lọc và tổng hợp theo trường:** mỗi dòng là một JSON nên hệ thống log
>    (Render logs, Datadog...) tách được `user_id`, `cost_usd` → trả lời được
>    "user nào tiêu nhiều tiền nhất hôm nay" bằng cách group theo `user_id` và
>    cộng `cost_usd`. `print("đã trả lời xong")` không có user, không có số liệu.
> 2. **Đo theo thời gian và đặt cảnh báo:** có `timestamp` ISO/UTC và `level` →
>    đếm số `ask_completed` hoặc số log `error` trong 5 phút để vẽ biểu đồ
>    traffic/tỷ lệ lỗi, hoặc cảnh báo khi `tokens_in` tăng bất thường.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo thật trên máy tôi (build bằng Dockerfile gốc của đề và Dockerfile đã sửa):
>
> | Bản | Dung lượng |
> |-----|-----------|
> | 1 stage (bản đầu, `python:3.11`) | 1730 MB (1.73GB) |
> | Multi-stage (`python:3.11-slim`) | 271 MB |
>
> Chênh lệch ~1.46GB chủ yếu là base image: `python:3.11` bản đầy đủ dựa trên
> Debian kèm bộ công cụ build (gcc, make, các gói header `-dev`, git...) mà app
> chạy không cần. `docker history agent:single` cho thấy layer `pip install` chỉ
> ~95MB, phần lớn còn lại là của base. Bản multi-stage dùng `slim` và chỉ copy
> thư mục `/install` (thư viện đã cài) từ stage builder sang, không mang theo
> công cụ build hay cache của pip. Bản 1 stage còn `COPY . .` nên kéo cả những
> file không cần vào image.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi thêm một dòng comment vào `app/main.py` rồi build lại với `--progress=plain`:
>
> - **Dùng lại cache:** `WORKDIR /app`, `COPY requirements.txt`,
>   `RUN pip install ...` (stage builder) và `COPY --from=builder /install` đều
>   báo `CACHED`, vì `requirements.txt` không đổi.
> - **Chạy lại:** từ `COPY app ./app` trở đi (`COPY utils`, `COPY requirements.txt`,
>   `RUN useradd ... chown`), chỉ mất khoảng 1–2 giây.
>
> Với Dockerfile gốc đặt `COPY . .` trước `RUN pip install`, tôi build thử sau
> cùng thay đổi đó: layer `COPY . .` đổi → cache bị hủy từ đó trở đi → `pip install`
> chạy lại từ đầu (log hiện `Collecting fastapi>=0.110...`, tải lại toàn bộ thư
> viện). Nghĩa là sửa một ký tự code cũng phải cài lại dependency, build, CI và
> deploy đều chậm theo.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi chạy root:
> 1. Code Python có lỗ hổng (vd. một thư viện bị RCE, hoặc lỡ `eval` input của
>    người dùng) → kẻ tấn công chạy được lệnh trong container.
> 2. Process đó là **root (uid 0)** → đọc/ghi mọi file trong container, cài thêm
>    công cụ, đọc secret trong biến môi trường.
> 3. uid 0 trong container cũng là uid 0 với kernel của host (nếu không bật user
>    namespace). Chỉ cần một cấu hình lỏng — mount `/var/run/docker.sock`, mount
>    thư mục host, `--privileged` — hoặc một lỗ hổng kernel/runtime để thoát
>    container, là root trong container thành root trên host.
>
> `USER appuser` (uid 10001 — tôi kiểm tra `docker run ... id` ra đúng
> `uid=10001(appuser)`) cắt chuỗi ở bước 2: kẻ tấn công chỉ có quyền user thường,
> không cài được gói hệ thống, không ghi được ngoài `/app`, và nếu có thoát ra
> host thì cũng chỉ là một uid không có đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt: gửi 10 request lúc 10:00:59
> (vẫn thuộc phút 10:00, bộ đếm còn trống → cả 10 đều qua), đến 10:01:00 bộ đếm
> reset về 0, gửi tiếp 10 request trong 10:01:00–10:01:01 → cũng qua hết. Mỗi
> "phút đồng hồ" đều đúng luật 10/phút, nhưng thực tế là 20 request dồn trong
> khoảng 2 giây, gấp đôi hạn mức.
>
> Với sliding window, request lúc 10:01:00 vẫn thấy 10 request trong 60 giây gần
> nhất nên bị 429. Tôi thấy đúng điều này khi test: gọi 15 lần liên tiếp thì 10
> lần đầu 200, 5 lần sau 429 kèm `Retry-After: 60`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - **Rate limit** giới hạn **số request trong 60 giây** — chống spam/DoS, trả
>   **429**, tự hết khi cửa sổ trôi qua.
> - **Cost guard** giới hạn **tổng tiền trong tháng** của mỗi user — bảo vệ ngân
>   sách, trả **402**, chỉ hết khi sang tháng mới.
>
> **Rate limit cho qua nhưng cost guard chặn:** user gửi đều 5 request/phút (dưới
> hạn 10/phút) nhưng mỗi câu hỏi dài vài nghìn token, suốt cả tháng → tổng chi phí
> vượt `MONTHLY_BUDGET_USD` → 402. Tôi thử bằng container có ngân sách `0.00003`
> USD: 2 request đầu 200, request thứ 3 đã bị 402 dù mới gọi 3 lần.
>
> **Cost guard cho qua nhưng rate limit chặn:** một script gửi 11 câu "hi" trong
> vài giây → mỗi câu chỉ tốn ~0.00002 USD, ngân sách gần như còn nguyên, nhưng
> request thứ 11 vẫn bị 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Giả sử một endpoint duy nhất vừa làm liveness vừa ping Redis:
> 1. Redis mất kết nối → cả 3 container cùng trả 503 cho health check.
> 2. Sau vài lần check fail liên tiếp (vd. `retries: 5`, 10s/lần), orchestrator
>    kết luận cả 3 container "chết" và **restart cả 3** — dù process Python vẫn
>    khỏe, lỗi nằm ở Redis.
> 3. Trong lúc restart, không còn instance nào nhận request; request đang xử lý
>    bị cắt ngang.
> 4. Container khởi động lại nhưng health check vẫn fail vì Redis chưa về → bị
>    restart tiếp (crash loop), có thể bị backoff lâu hơn cả 30 giây Redis mất.
> 5. Redis về rồi, các container vẫn phải khởi động xong và qua health check mới
>    nhận traffic → gián đoạn dài hơn nhiều so với sự cố gốc.
>
> Khi tách ra: `/health` vẫn 200 nên không container nào bị restart, chỉ `/ready`
> trả 503 → load balancer tạm ngừng đẩy traffic, Redis về thì `/ready` 200 ngay.
> Tôi đã thử `docker compose stop redis`: `/health` 200, `/ready` 503
> `{"status":"not ready","redis":false}`; bật Redis lại thì `/ready` về 200.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Compose của tôi map cố định `8000:8000` nên `--scale agent=3` bị trùng cổng;
> tôi chạy 3 instance ở cổng 8000/8001/8002 dùng chung Redis và gọi xen kẽ với
> cùng `X-User-Id`. `history_length` lần lượt: 0 (:8000) → 2 (:8001) → 4 (:8002)
> → 6 (:8000) → 8 (:8001) → 10 (:8002) — tăng đều 2 mỗi lượt (1 câu hỏi + 1 câu
> trả lời) dù mỗi lần vào một container khác, vì lịch sử nằm trong Redis.
>
> Nếu dùng dict Python, mỗi container có dict riêng: con số phụ thuộc container
> nào nhận request — lượt 1–3 đều ra 0 (mỗi container lần đầu gặp user), lượt 4–6
> đều ra 2, tức là agent "quên" những gì đã nói ở container khác. Container bị
> restart hoặc redeploy thì mất sạch lịch sử.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi:** deploy Render thành công, `/health` 200, `/ready` 200
> `{"status":"ready","redis":true}`, nhưng gọi `/ask` kèm khóa của tôi vẫn trả
> `401 {"detail":"invalid or missing API key"}`.
>
> **Tìm nguyên nhân:** `/ready` 200 nghĩa là app và Redis đều ổn, nên vấn đề chỉ
> nằm ở bước so khóa. Tôi thử cả khóa định dùng cho cloud lẫn khóa trong `.env`
> local: cả hai đều 401 → giá trị `AGENT_API_KEY` trên Render không khớp khóa nào,
> khả năng cao lúc dán vào ô env của Blueprint bị dính ký tự thừa hoặc sai giá trị.
> Code so bằng `compare_digest`, khớp tuyệt đối, nên thừa một dấu cách cũng sai.
>
> **Sửa:** vào service `day12-agent` → Environment → sửa `AGENT_API_KEY`, dán lại
> đúng chuỗi, Save để Render redeploy (dashboard hiện deploy thứ hai, trigger
> Manual). Sau đó `/ask` có key trả 200, gọi 15 lần thì 10 lần 200 và 5 lần 429,
> `pytest tests/test_cp5.py` pass 9/9.
>
> (Lỗi phụ ở local trước đó: `docker stop` mất 10 giây rồi exit 137 vì
> `CMD ["sh","-c","uvicorn ..."]` để `sh` làm PID 1, không chuyển SIGTERM cho
> uvicorn; sửa bằng `exec uvicorn ...`, giờ dừng trong 1 giây, exit 0.)
