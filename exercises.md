# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder `*Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Le Thi Cham Anh  Mã học viên: 2A202602846

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, service `agent` là service mới tạo nên lúc đầu chưa có
> biến nào. Nếu tôi quên set `AGENT_API_KEY` mà code có mặc định `"changeme"`,
> app vẫn khởi động bình thường, `/health` vẫn trả 200, dashboard hiện màu xanh.
> Nhưng URL của service là công khai, và `"changeme"` là giá trị ai cũng đoán
> được (nó còn nằm trong repo public). Bất kỳ ai cũng gọi được `/ask` bằng khóa
> đó và tiêu ngân sách LLM của tôi, còn tôi thì không hề biết.
>
> Khi không có mặc định, tôi thử chạy thiếu biến và app dừng ngay lúc khởi động:
> `ValidationError: 1 validation error for Settings — agent_api_key: Field required`.
> Deploy sẽ fail healthcheck và log chỉ thẳng ra biến nào bị thiếu. Lỗi hiện ra
> ngay trong 1 phút lúc deploy, chứ không phải vài tuần sau khi hóa đơn đã tăng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật khi tôi gọi `/ask` lần thứ 3 với cùng user:
>
> ```
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T17:58:48.148664+00:00", "user_id": "sv-test", "tokens_in": 94, "tokens_out": 47, "cost_usd": 4.23e-05}
> ```
>
> 1. **Lọc và truy vết theo user**: vì `user_id` là một trường riêng, tôi có thể
>    lọc (ví dụ `jq 'select(.user_id=="sv-test")'` hoặc ô tìm kiếm log trên
>    Railway) để xem toàn bộ request của một người khi họ báo lỗi. Với
>    `print("đã trả lời xong")` thì không biết dòng đó là của ai.
> 2. **Tính toán số liệu**: cộng `cost_usd` theo ngày/user để biết chi phí, hoặc
>    theo dõi `tokens_in` tăng dần (3 → 43 → 94 trong 3 lần gọi của tôi, vì
>    history dài thêm) để đặt cảnh báo khi prompt phình quá lớn. Ngoài ra
>    `timestamp` có múi giờ UTC nên log từ nhiều container ghép lại được đúng
>    thứ tự.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản 1 stage lấy từ Dockerfile gốc của repo (`FROM python:3.11`, `COPY . .`,
> `pip install`). Chênh lệch khoảng 1.4 GB gồm:
>
> - **Base image đầy đủ `python:3.11`** (dựa trên Debian đầy đủ): có sẵn gcc,
>   make, header file để biên dịch, git, curl, nhiều thư viện hệ thống... Đây
>   là phần lớn nhất. Bản multi-stage dùng `python:3.11-slim` nên không có những
>   thứ này.
> - **Rác từ quá trình cài đặt**: bản 1 stage giữ lại cache của pip trong cùng
>   layer. Bản multi-stage cài vào `/opt/venv` ở stage `builder` (có
>   `PIP_NO_CACHE_DIR=1`), rồi runtime chỉ `COPY --from=builder /opt/venv`, nên
>   mọi thứ khác của builder bị bỏ lại.
> - **`COPY . .` copy cả thư mục**: bản multi-stage chỉ copy `app/` và `utils/`
>   (và có `.dockerignore`), không kéo theo `tests/`, `.git`, tài liệu...
>
> Image nhỏ hơn thì pull nhanh hơn khi deploy/scale, và ít phần mềm thừa hơn
> nên ít lỗ hổng bảo mật hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi đổi `SERVICE_VERSION = "1.0.0"` thành `"1.0.1"` rồi build lại với
> `--progress=plain`. Kết quả:
>
> - **CACHED**: `RUN python -m venv`, `COPY requirements.txt .`,
>   `RUN pip install -r requirements.txt`, `RUN groupadd/useradd`,
>   `WORKDIR /app`, `COPY --from=builder /opt/venv`.
> - **Chạy lại**: `COPY app/ ./app/` (vì file trong `app/` đổi) và
>   `COPY utils/ ./utils/` (vì nó nằm sau layer vừa bị đổi, cache bị vô hiệu
>   từ đó trở xuống). Hai layer này chỉ mất 0.1s mỗi cái.
>
> Nếu đặt `COPY . .` trước `RUN pip install`, thì chỉ cần sửa 1 ký tự trong code
> là checksum của layer COPY thay đổi → mọi layer phía sau, kể cả `pip install`,
> phải chạy lại. Mỗi lần sửa code sẽ phải tải và cài lại toàn bộ thư viện
> (fastapi, pydantic, redis, uvicorn...), tốn vài chục giây đến vài phút, thay
> vì chưa tới 1 giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> 1. Code có lỗ hổng cho phép thực thi lệnh (ví dụ một thư viện bị lỗi
>    deserialize, hoặc đoạn code đưa input của user vào `subprocess`/`eval`).
> 2. Kẻ tấn công chạy được shell trong container. Nếu container chạy bằng root,
>    shell đó có uid 0: đọc/ghi được mọi file trong container, sửa code và
>    thư viện trong `/opt/venv` để cài backdoor, cài thêm công cụ.
> 3. Uid 0 trong container cũng chính là uid 0 trên host (nếu không bật user
>    namespace remap). Nên chỉ cần thêm một lỗ hổng thoát container (lỗi
>    kernel/runc) hoặc một cấu hình sai như mount `/var/run/docker.sock` hay thư
>    mục của host vào container, kẻ tấn công trở thành root trên máy host.
>
> Lệnh `USER app` cắt chuỗi ở bước 2: shell của kẻ tấn công chỉ là user `app`
> không có đặc quyền. Nó không sửa được file hệ thống, không ghi được vào
> `/opt/venv` (thuộc root), không cài được gói, và phần lớn các kỹ thuật thoát
> container đều cần root ngay từ đầu. Kể cả thoát được thì trên host nó cũng chỉ
> là một uid thường.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request** trong 2 giây.
>
> Cách làm: gửi 10 request lúc 10:00:59 → hợp lệ vì bộ đếm của phút 10:00 mới là
> 10. Sang 10:01:00 bộ đếm reset về 0, gửi tiếp 10 request → vẫn hợp lệ. Trong
> khoảng 10:00:59–10:01:00 server nhận 20 request, gấp đôi hạn mức.
>
> Sliding window của tôi lưu timestamp từng request trong Redis sorted set và
> đếm số request trong 60 giây **tính ngược từ thời điểm hiện tại**. Lúc
> 10:01:00, 10 request lúc 10:00:59 vẫn nằm trong cửa sổ nên request thứ 11 bị
> trả 429. Mọi khoảng 60 giây bất kỳ đều không quá 10 request.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - **Rate limit** giới hạn **tốc độ**: số request trong 60 giây gần nhất (10
>   request/phút), trả **429**. Nó bảo vệ server khỏi bị spam/quá tải, và
>   hết hạn nhanh (chờ 1 phút là gọi lại được).
> - **Cost guard** giới hạn **tổng tiền**: tổng `cost_usd` của user trong tháng
>   (10 USD), trả **402**. Nó bảo vệ ví tiền, và chỉ reset sang tháng sau.
>
> **Rate limit cho qua, cost guard chặn**: một user dùng đều đặn 5 request/phút
> suốt nhiều ngày, mỗi câu hỏi rất dài nên mỗi lần tốn nhiều token. Không bao
> giờ vượt 10/phút, nhưng cộng dồn chạm 10 USD vào ngày 20 → từ đó bị 402 cho
> tới hết tháng.
>
> **Cost guard cho qua, rate limit chặn**: một user mới (chi phí tháng này gần
> 0) chạy script gửi 15 request trong 5 giây. Tổng chi phí chỉ khoảng vài phần
> nghìn USD (mỗi lần gọi tôi đo được khoảng 0.00002–0.00004 USD), nhưng request
> thứ 11 bị 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối. Cả 3 container cùng lúc bắt đầu trả lỗi ở endpoint gộp
>    (vì cả 3 cùng phụ thuộc một Redis).
> 2. Orchestrator (Docker healthcheck / Railway / Kubernetes liveness) gọi probe
>    mỗi 15 giây, sau vài lần fail liên tiếp (retries = 3) thì đánh dấu **cả 3**
>    container là unhealthy.
> 3. Nó coi container "chết" và **restart cả 3 cùng lúc**. Những request đang
>    xử lý dở bị cắt ngang, và trong lúc khởi động lại thì không còn instance
>    nào nhận traffic → toàn bộ service sập, kể cả những request không cần Redis.
> 4. Restart không sửa được Redis, nên nếu Redis chưa về thì container mới cũng
>    fail probe → restart tiếp → vòng lặp restart có backoff.
> 5. Redis về sau 30 giây, nhưng các container có thể đang ở giữa lần restart
>    hoặc đang chờ backoff → service sập lâu hơn nhiều so với 30 giây.
>
> Khi tách riêng: `/health` không gọi Redis nên vẫn trả 200 → không container
> nào bị restart. Chỉ `/ready` trả 503 → load balancer tạm ngừng gửi traffic
> vào. Khi Redis về, `/ready` trả 200 lại và traffic chạy tiếp ngay, không mất
> thời gian khởi động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi chạy 3 container `agent` + 1 Redis, rồi gọi `/ask` 6 lần với cùng
> `X-User-Id: sv-scale`, lần lượt xoay vòng qua 3 container (172.25.0.3 → .4 →
> .5 → .3 → .4 → .5). Kết quả:
>
> ```
> call 1 -> 172.25.0.3: history_length=0
> call 2 -> 172.25.0.4: history_length=2
> call 3 -> 172.25.0.5: history_length=4
> call 4 -> 172.25.0.3: history_length=6
> call 5 -> 172.25.0.4: history_length=8
> call 6 -> 172.25.0.5: history_length=10
> ```
>
> Log cho thấy mỗi container xử lý đúng 2 request, nhưng `history_length` vẫn tăng
> đều thêm 2 mỗi lần (1 câu hỏi + 1 câu trả lời), vì cả 3 cùng đọc/ghi chung một
> Redis.
>
> Nếu lưu trong dict Python, mỗi container có dict riêng trong bộ nhớ của nó.
> Con số sẽ nhảy lung tung theo container nhận request: 0, 0, 0, 2, 2, 2 (mỗi
> container chỉ nhớ phần hội thoại nó từng xử lý). Agent sẽ "quên" những gì user
> vừa nói nếu request rơi vào container khác. Và khi container restart hoặc
> redeploy thì toàn bộ lịch sử bị mất.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi**: tôi tạo project trên Railway, chạy `railway add --database redis` rồi
> chạy `railway up`. Build log báo thành công (`pip install` xong,
> `exporting to docker image format`, `image push`), nhưng trên dashboard không
> hề có service nào cho app, chỉ có Redis. Và Redis bị treo ở trạng thái
> "Deploying", `replicas: 0/1 running`.
>
> **Tìm nguyên nhân**: tôi chạy `railway status` và `railway service list`, thấy
> dòng `Redis (linked)`, nghĩa là terminal đang link với service Redis (do
> lệnh `railway add --database redis` tự link vào service vừa tạo). Chạy
> `railway deployment list --service Redis` thấy có 2 deployment: bản
> `redis:8.2` gốc và một bản mới hơn, chính là code app. Vậy `railway up` đã
> **deploy code app đè lên service Redis**.
>
> **Sửa**: gỡ bản deploy nhầm bằng `railway down --service Redis -y`, sau đó
> redeploy Redis từ image `redis:8.2` trên dashboard và chờ nó Online lại. Tạo
> service riêng bằng `railway add --service agent`, set biến bằng
> `railway variables --service agent --set ...` (trong đó
> `REDIS_URL=${{Redis.REDIS_URL}}` để tham chiếu sang service Redis), và từ đó
> luôn thêm `--service agent` khi chạy `railway up` để không deploy nhầm nữa.
>
> Ngoài ra tôi phát hiện `startCommand` trong `railway.toml` dùng
> `--port $PORT`. Railway chạy lệnh này không qua shell nên `$PORT` sẽ không
> được thay thành số. Tôi bỏ `startCommand` để Railway dùng `CMD` trong
> Dockerfile (`sh -c "... --port ${PORT:-8000}"`). Trong lúc sửa còn gặp sự cố
> phía Railway (`operation timed out` và `Deploys have been paused temporarily`
> do sự cố "API degradation" của họ). Lỗi này không phải do code, nên chỉ cần
> chờ hệ thống hồi phục rồi deploy lại.
