# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder "Câu trả lời của bạn" bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Hữu Đức  Mã học viên: 2A202602459

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway lần đầu, tôi tạo service mới và rất dễ quên set
> `AGENT_API_KEY` trong tab Variables. Nếu `Settings` có mặc định `"changeme"`,
> service vẫn build xong, `/health` vẫn trả 200, Railway báo "Deploy complete"
> và tôi tưởng mọi thứ ổn. Nhưng thực chất URL public đang được bảo vệ bằng một
> khóa mà ai đọc repo (public) cũng biết — bất kỳ ai gọi `/ask` với
> `X-API-Key: changeme` đều tiêu ngân sách của tôi, và tôi chỉ phát hiện khi
> thấy hóa đơn hoặc log lạ. Với `agent_api_key: str` không có mặc định,
> `Settings()` ném `ValidationError` ngay lúc khởi động, container crash, health
> check fail và deploy bị đánh dấu thất bại — lỗi hiện ra trong vài giây ở
> đúng chỗ (log deploy ghi rõ thiếu field `agent_api_key`), trước khi có
> request nào tới. Chết sớm biến một lỗ hổng bảo mật im lặng thành một lỗi
> cấu hình ồn ào, dễ sửa.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log thu được (`docker compose logs agent`, 2026-09-29):

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:12:30.319987+00:00", "user_id": "sv-scale-918", "tokens_in": 250, "tokens_out": 46, "cost_usd": 6.51e-05}
```

> 1. **Lọc và tổng hợp theo trường.** Vì mỗi dòng là JSON có key cố định, tôi
>    lọc được đúng request của một user rồi cộng chi phí:
>
>    ```bash
>    docker compose logs agent --no-log-prefix | grep '^{' | python3 -c "import sys,json; r=[json.loads(l) for l in sys.stdin]; a=[x for x in r if x['event']=='ask_completed' and x.get('user_id')=='sv-scale-918']; print(len(a), 'request, tong', round(sum(x['cost_usd'] for x in a),7), 'USD')"
>    # => 6 request, tong 0.0002754 USD
>    ```
>
>    Chính cách lọc theo `user_id` này là cách tôi đếm được mỗi container
>    agent-1/2/3 xử lý bao nhiêu request ở câu 9. Với `print("đã trả lời xong")`
>    thì không biết của user nào, tốn bao nhiêu token, không có gì để cộng.
> 2. **Đặt cảnh báo và dựng dashboard tự động.** Công cụ log của platform
>    (Railway log explorer, Loki, CloudWatch, Datadog...) parse JSON thành
>    trường, nên tôi tạo được biểu đồ `tokens_in`/`cost_usd` theo thời gian,
>    hoặc alert khi `level == "error"` hay một `user_id` vượt X USD/giờ. Có
>    `timestamp` ISO-8601 UTC nên sắp xếp và ghép log từ 3 container theo đúng
>    thứ tự thời gian được, kể cả khi container ở múi giờ khác nhau.

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
| 1 stage (bản đầu) | 1.73 GB (1728 MB) |
| Multi-stage | 271 MB |

Số đo từng layer (`docker history`, 2026-09-29):

- 1 stage (`FROM python:3.11`, `COPY . .`, `pip install` không `--no-cache-dir`):
  các layer base image `python:3.11` gồm 694 MB, 202 MB, 134 MB, 70.6 MB, 65 MB,
  19.9 MB; layer `pip install` 95.1 MB; layer `COPY . .` 586 kB.
- Multi-stage (`python:3.11-slim`): layer `COPY /install /usr/local` 65.5 MB;
  `COPY app/` 98.3 kB; `COPY utils/` 28.7 kB; `useradd` 184 kB.

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch khoảng 1.46 GB (1728 MB so với 271 MB). Cộng các layer theo
> `docker history` thì bản 1 stage là 1282 MB, bản multi-stage là 207 MB
> (`docker images` báo lớn hơn vì Docker Desktop dùng containerd image store,
> tính thêm cả blob nén của layer — nhưng tỉ lệ vẫn giống nhau). Phần chênh
> gồm:
>
> 1. **Base image đầy đủ `python:3.11` (~1185 MB layer)** so với
>    `python:3.11-slim`: bản đầy đủ dựa trên Debian `buildpack-deps`, mang theo
>    gcc/g++, make, header phát triển (`libssl-dev`, `libpq-dev`...), git,
>    curl, ImageMagick... — những thứ chỉ cần khi *biên dịch* package, không
>    cần khi *chạy* app. Đây là phần lớn nhất (694 MB + 202 MB + 134 MB ...).
> 2. **Cache của pip**: bản đầu chạy `pip install` không có `--no-cache-dir`,
>    nên layer 95.1 MB chứa cả file wheel đã tải trong `~/.cache/pip`. Bản
>    multi-stage chỉ copy thư mục `/install` (65.5 MB) — đúng các package đã
>    cài, không có cache, không có công cụ build.
> 3. **`COPY . .`** kéo cả `tests/`, tài liệu `.md`, `grade.py`... vào image
>    (586 kB) — nhỏ, nhưng là thứ không cần ở production; bản mới chỉ copy
>    `app/` và `utils/` (~127 kB).
>
> Multi-stage giúp được vì stage `builder` có thể "bẩn" bao nhiêu cũng được,
> image cuối chỉ lấy kết quả `COPY --from=builder`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi sửa 1 ký tự trong docstring của hàm `ask` (`"""Hỏi agent một câu."""` →
> `!`) rồi build lại bằng `docker build --progress=plain` (2026-09-29):
>
> | Layer (Dockerfile multi-stage của tôi) | Kết quả |
> |---|---|
> | `[builder 2/4] WORKDIR /app` | CACHED |
> | `[builder 3/4] COPY requirements.txt .` | CACHED |
> | `[builder 4/4] RUN pip install ...` | CACHED |
> | `[stage-1 3/6] COPY --from=builder /install /usr/local` | CACHED |
> | `[stage-1 4/6] COPY app/ ./app/` | chạy lại (0.0s) |
> | `[stage-1 5/6] COPY utils/ ./utils/` | chạy lại (0.0s) |
> | `[stage-1 6/6] RUN useradd ... && chown -R ...` | chạy lại (0.2s) |
>
> Docker so checksum nội dung file được COPY: `requirements.txt` không đổi nên
> layer `pip install` và mọi layer trước nó dùng lại cache. `app/` đổi nên từ
> layer `COPY app/` trở đi cache bị vô hiệu, kể cả `COPY utils/` và `useradd`
> dù chúng không đổi — cache là một chuỗi, gãy một mắt thì các mắt sau phải
> chạy lại. Tổng thời gian build lại chỉ khoảng 0.2 giây.
>
> Với Dockerfile gốc 1 stage (`COPY . .` rồi `RUN pip install`), cùng thay đổi
> đó làm layer `COPY . .` đổi checksum, kéo theo **`pip install` chạy lại toàn
> bộ, mất 15.4 giây** (tải và cài lại mọi thư viện) dù `requirements.txt`
> không đổi một chữ. Mỗi lần sửa code là một lần cài lại dependency — chậm
> hơn nhiều khi làm việc và khi CI build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi container chạy bằng root:
>
> 1. Code có lỗ hổng cho phép thực thi lệnh — ví dụ dùng `eval`/`pickle` trên
>    input của người dùng, hoặc một thư viện trong `requirements.txt` có CVE
>    remote code execution.
> 2. Kẻ tấn công có shell chạy với quyền của process uvicorn — tức **uid 0**
>    trong container.
> 3. Là root, họ ghi được mọi nơi trong container: sửa file trong
>    `/usr/local/lib/python3.11` để cài backdoor, cài thêm công cụ bằng
>    `apt-get`, đọc mọi secret trong biến môi trường và file được mount.
> 4. Container chỉ là process được cô lập bằng namespace, dùng chung kernel với
>    host; uid 0 trong container mặc định **chính là uid 0 trên host** (không
>    bật user namespace remap). Chỉ cần thêm một điểm yếu thoát container — lỗ
>    hổng kernel, container chạy `--privileged`, mount `/var/run/docker.sock`
>    hoặc thư mục của host — là họ thành root trên máy host, điều khiển được
>    các container khác.
>
> `USER appuser` (uid 10001) cắt chuỗi ở **bước 2**: shell của kẻ tấn công chỉ
> là một user thường. Họ không ghi được vào `/usr/local` hay thư mục hệ thống,
> không cài được package, không dùng được các capability của root; và nếu thoát
> được ra ngoài thì họ cũng chỉ là uid 10001 — một user không có quyền gì trên
> host. Lưu ý: trong Dockerfile của tôi `/app` vẫn được `chown` cho `appuser`,
> nên kẻ tấn công vẫn sửa được code app trong container đang chạy; muốn chặt
> hơn có thể để `/app` thuộc root và chỉ cho `appuser` quyền đọc.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request** (gấp đôi hạn mức). Cách làm:
>
> - Lúc `10:00:59` (giây cuối của phút 10:00), gửi liên tiếp 10 request →
>   bộ đếm phút 10:00 lên 10/10, tất cả được cho qua.
> - Lúc `10:01:00`, bộ đếm reset về 0 vì đã sang phút mới → gửi tiếp 10
>   request, cũng được cho qua.
>
> Trong khoảng 2 giây (thậm chí chỉ vài trăm mili giây quanh mốc giây 00),
> server nhận 20 request dù "hạn mức" là 10/phút.
>
> Sliding window của tôi không bị lỗi này: `RateLimiter.check` xóa các mục cũ
> hơn `now - 60` khỏi sorted set `ratelimit:<user_id>` rồi đếm `ZCARD`. Lúc
> `10:01:00`, 10 request của `10:00:59` vẫn nằm trong 60 giây gần nhất nên
> `count = 10 >= limit` → request thứ 11 bị 429. Ở bất kỳ cửa sổ 60 giây nào
> cũng không bao giờ lọt quá 10 request.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> | | Rate limit | Cost guard |
> |---|---|---|
> | Đo cái gì | **Số request** trong 60 giây gần nhất | **Tổng tiền (USD)** đã tiêu trong tháng |
> | Lưu ở đâu | Sorted set `ratelimit:<user>`, hết hạn sau 60s | Số thực `cost:<user>:<YYYY-MM>`, giữ ~40 ngày |
> | Bảo vệ khỏi | Spam, script chạy vòng lặp, burst làm quá tải server | Hóa đơn LLM vượt ngân sách |
> | Mã lỗi | 429 Too Many Requests (thử lại sau `Retry-After`) | 402 Payment Required (chờ sang tháng) |
>
> Rate limit giới hạn **tốc độ**, cost guard giới hạn **tổng lượng**. Một cái
> không thay được cái kia.
>
> **Rate limit cho qua nhưng cost guard chặn:** một user viết script gọi `/ask`
> đều đặn đúng 10 request/phút — không bao giờ vượt hạn mức nên không dính 429.
> Nhưng chạy suốt ngày đêm thì 10 × 60 × 24 = 14.400 request/ngày; với chi phí
> đo được khoảng 6.5e-05 USD/request (log ở câu 2, lịch sử càng dài càng đắt)
> thì mỗi ngày tốn khoảng 0.94 USD, tới khoảng ngày 11 thì vượt 10 USD
> `MONTHLY_BUDGET_USD` → cost guard trả 402.
>
> **Cost guard cho qua nhưng rate limit chặn:** một user mới, cả tháng mới tiêu
> 0.001 USD, bấm gửi 15 lần trong 5 giây (hoặc app client bị lỗi retry liên
> tục). Ngân sách còn gần như nguyên vẹn nên cost guard không chặn, nhưng
> request thứ 11 bị 429 vì đã đủ 10 request trong 60 giây. Tôi đã chạy lệnh số
> 5 trong `DEPLOYMENT.md` trên URL Railway với một user mới (gần như chưa tiêu
> đồng nào) và nhận đúng
> `200 200 200 200 200 200 200 200 200 200 429 429 429 429 429`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Giả sử orchestrator dùng endpoint gộp làm liveness probe (ví dụ Kubernetes
> mặc định: probe mỗi 10 giây, fail 3 lần liên tiếp thì restart):
>
> 1. **t = 0s**: Redis mất kết nối. Cả 3 container vẫn chạy bình thường,
>    process không có lỗi gì.
> 2. **t ≈ 0–10s**: probe gọi endpoint gộp → `ping()` Redis thất bại → cả 3
>    container **cùng lúc** trả 503. Load balancer thấy cả 3 đều hỏng, không
>    còn backend nào → mọi request của người dùng nhận 502/503.
> 3. **t ≈ 30s**: probe fail lần thứ 3 → orchestrator kết luận process "chết"
>    và **kill + restart cả 3 container** — dù process hoàn toàn khỏe và lỗi
>    nằm ở Redis. Restart không sửa được Redis.
> 4. **t ≈ 30s+**: Redis có lại, nhưng 3 container đang khởi động lại (tải
>    image, import thư viện, chạy `lifespan`) — request đang xử lý dở bị cắt
>    ngang, cluster vẫn không phục vụ được ai. Nếu container mới lên mà Redis
>    còn chập chờn, probe lại fail → restart tiếp → rơi vào vòng
>    CrashLoopBackOff với thời gian chờ tăng dần.
> 5. **Kết quả**: một sự cố Redis 30 giây biến thành downtime toàn cụm lâu hơn
>    nhiều, và log đầy restart làm che mất nguyên nhân thật.
>
> Khi tách riêng như code của tôi: `/health` chỉ kiểm tra process (không gọi
> Redis) nên vẫn 200 → không container nào bị restart; `/ready` trả 503 → load
> balancer tạm rút 3 container khỏi vòng xoay (hoặc request `/ask` trả lỗi
> nhanh). Khi Redis có lại, `/ready` về 200 ngay ở lần probe kế tiếp và traffic
> quay lại — không mất thời gian khởi động lại, downtime chỉ khoảng 30 giây
> đúng bằng sự cố.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Kết quả đo (`docker compose up -d --scale agent=3`, nginx round-robin, 6 lần
gọi `/ask` với cùng `X-User-Id: sv-scale-918`, 2026-09-29):

| Lần gọi | 1 | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|
| `history_length` | 0 | 2 | 4 | 6 | 8 | 10 |

Container xử lý (đếm dòng `ask_completed` trong `docker compose logs agent`):
agent-1 = 3 request, agent-2 = 1 request, agent-3 = 2 request.

> Với Redis, `history_length` tăng đều 0 → 2 → 4 → 6 → 8 → 10 (mỗi lần `/ask`
> ghi thêm 2 message: `user` và `assistant`), dù 6 request rơi vào cả 3
> container khác nhau. Lý do: container nào cũng đọc/ghi cùng một key
> `history:sv-scale-918` trong Redis, nên process nào nhận request cũng thấy
> đủ lịch sử.
>
> Nếu lịch sử nằm trong một dict Python, mỗi container có một dict riêng trong
> RAM của nó. Với đúng thứ tự phân phối đã quan sát, mỗi container chỉ thấy
> những lượt nó tự xử lý, nên con số sẽ **nhảy lung tung và nhỏ hơn thật**:
> lần đầu một container gặp user này nó trả 0, lần tiếp theo trên cùng
> container đó trả 2, rồi 4... Ví dụ agent-1 xử lý 3 request thì trả
> 0, 2, 4; agent-3 xử lý 2 request trả 0, 2; agent-2 chỉ 1 request trả 0 —
> xen kẽ nhau thành một chuỗi kiểu 0, 0, 2, 0, 4, 2 thay vì 0…10. Lịch sử đầy
> đủ không nằm ở đâu cả, agent "quên" hội thoại tùy request rơi vào đâu; khi
> một container restart hoặc bị scale down thì phần lịch sử của nó mất hẳn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Thông báo lỗi** (Railway build log, deploy bằng `railway up` trong WSL):
>
> ```
> [internal] load build context: rpc error: code = Internal desc = ...
> header key "exclude-patterns" contains value with non-printable ASCII characters
> Build Failed: build daemon returned an error
> ```
>
> **Tìm nguyên nhân:**
>
> 1. "exclude-patterns" là danh sách pattern đọc từ `.dockerignore`, nên tôi
>    nghi file này. Bản đầu có comment tiếng Việt có dấu → xóa comment. Deploy
>    lại: **vẫn lỗi y hệt**.
> 2. Thấy `.gitignore` cũng có comment tiếng Việt và dấu `—` → đổi sang không
>    dấu, kiểm tra bằng `grep -P '[^\x09\x0a\x20-\x7e]'` là cả hai file sạch.
>    Deploy lại: **vẫn lỗi**.
> 3. Thử loại trừ: tạm đổi tên `.dockerignore` rồi deploy. Lần này lỗi khác
>    hẳn: `ERROR: Invalid requirement: '\x00\x00\x00...' (from line 1 of
>    requirements.txt)`. Tức là `requirements.txt` lên tới Railway toàn byte 0.
> 4. Kết luận: Railway CLI chạy trong WSL đọc file nằm trên ổ Windows
>    (`/mnt/d/...`) thì nhận được nội dung toàn `\x00` — `.dockerignore` cũng
>    vậy, và byte 0 chính là "non-printable ASCII" trong header. Comment tiếng
>    Việt chưa bao giờ là nguyên nhân. `cat`/`grep` trong WSL vẫn đọc file bình
>    thường, nên lỗi chỉ nằm ở cách CLI đọc file trên ổ mount từ Windows.
>
> **Cách sửa:** copy project sang filesystem riêng của WSL rồi deploy từ đó,
> chỉ định rõ project/environment/service:
>
> ```bash
> rsync -a --exclude .venv --exclude .git --exclude .env \
>   /mnt/d/AI_in_Action/day12/K4-L3B-Cloud-Service-And-Deployment/ ~/day12-deploy/
> cd ~/day12-deploy && railway up -p <project-id> -e <env-id> -s <service-id>
> ```
>
> Build thành công. Ngoài ra, khi kiểm tra `railway variables` tôi thấy service
> agent chưa có `REDIS_URL` (app sẽ rơi về mặc định `redis://localhost:6379/0`
> và `/ready` trả 503), nên đã set `REDIS_URL=${{Redis.REDIS_URL}}` trỏ sang
> service Redis. Kết quả: `/health` 200, `/ready` 200 `{"redis": true}`, `/ask`
> không key trả 401.
