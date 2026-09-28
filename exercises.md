# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Ngô Đức Chung  Mã học viên: 2A202602985

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: deploy lên Railway/Render, tôi quên thêm biến `AGENT_API_KEY` trong
> dashboard của platform trước khi ấn deploy. Vì `agent_api_key: str` không có
> default, `Settings()` ném `pydantic.ValidationError: field required` ngay ở
> `get_settings()`, container crash lúc khởi động, healthcheck fail liên tục và
> platform báo đỏ ngay trong log deploy — tôi biết thiếu biến trong vài giây.
>
> Nếu để mặc định `"changeme"`, app vẫn khởi động và healthcheck vẫn xanh bình
> thường, vì lỗi cấu hình không hề crash. Endpoint `/ask` vẫn public, và bất kỳ
> ai đoán ra (hoặc đọc được từ code mẫu trên GitHub) header
> `X-API-Key: changeme` là gọi được LLM miễn phí bằng ngân sách của tôi. Tôi chỉ
> phát hiện ra khi `cost_guard` báo vượt `monthly_budget_usd` hoặc khi nhận hóa
> đơn — tức là sau khi thiệt hại đã xảy ra, không phải lúc deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Chạy service ở local (`REDIS_URL=fake://`) và gọi `POST /ask` hai lần với
> cùng `X-User-Id: sv-test`, tôi thu được dòng log JSON sau trên stdout:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:08:30.420466+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
> ```
>
> Hai việc làm được mà `print` không làm được:
> 1. **Lọc/truy vấn theo trường**: đẩy log này vào một hệ thống đọc log dạng
>    JSON (CloudWatch, Datadog, `jq` ngay trên terminal) rồi lọc
>    `event == "ask_completed" and user_id == "sv-test"` hoặc
>    `cost_usd > 0.001` — vì mỗi field là một key có tên, máy đọc được ranh
>    giới giữa các giá trị. `print("đã trả lời xong")` chỉ là một chuỗi, muốn
>    lọc theo `user_id` phải viết regex đoán mò và dễ vỡ khi câu trả lời đổi.
> 2. **Tổng hợp/cảnh báo tự động**: cộng dồn `cost_usd` của tất cả dòng
>    `ask_completed` trong một khoảng thời gian để dựng dashboard chi phí theo
>    user, hoặc đặt cảnh báo "tổng `cost_usd` trong 1 giờ > X" — việc này cần
>    giá trị số có kiểu (`float`), còn chuỗi tự do thì không cách nào cộng lại
>    được một cách đáng tin cậy.

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

> *Câu trả lời của bạn*

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Trong `Dockerfile` hiện tại, thứ tự là: `COPY requirements.txt .` →
> `RUN pip install ...` (ở stage `builder`) → rồi mới `COPY app ./app` và
> `COPY utils ./utils` (ở stage `runtime`). Khi sửa một ký tự trong
> `app/main.py` rồi build lại: `FROM python:3.11-slim AS builder`,
> `COPY requirements.txt .` và `RUN pip install` **dùng lại cache** — Docker
> so sánh checksum `requirements.txt`, không đổi thì không cài lại. Chỉ có
> `COPY app ./app` trở đi (và `USER`, `HEALTHCHECK`, `CMD` không phải chạy lại
> vì chúng không tạo layer tốn thời gian) phải build lại vì nội dung thư mục
> `app` đã đổi.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`: bất kỳ thay đổi nào trong
> repo — kể cả một ký tự trong `app/main.py`, không liên quan gì tới
> `requirements.txt` — cũng làm layer `COPY . .` invalidate, và vì Docker
> cache theo thứ tự tuyến tính, layer `RUN pip install` đứng NGAY SAU nó cũng
> mất cache theo, dù danh sách thư viện không đổi một chữ nào. Kết quả: sửa
> code xong build lại tốn thêm hàng chục giây đến vài phút để cài lại toàn bộ
> dependency, thay vì gần như tức thì.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python của tôi có một lỗ hổng — ví dụ một thư viện
> deserialize dữ liệu không kiểm soát, hoặc một endpoint ghép chuỗi vào lệnh
> shell (command injection); (2) kẻ tấn công khai thác lỗ hổng đó và có được
> khả năng thực thi lệnh tùy ý **bên trong tiến trình** đang chạy `/ask`;
> (3) nếu tiến trình đó chạy bằng root (UID 0) như Dockerfile gốc, lệnh họ
> chạy cũng chạy với quyền root **trong container** — họ ghi/đọc mọi file
> trong container, cài thêm tool, đọc biến môi trường (kể cả
> `AGENT_API_KEY`); (4) từ đó họ tìm đường thoát container: khai thác lỗ hổng
> kernel cần quyền root để trigger, hoặc lợi dụng một volume/socket bị mount
> nhầm (ví dụ `/var/run/docker.sock`) — cả hai đường này đều CẦN họ đã là
> root trong container trước đã; (5) thoát được là họ có quyền root ngay trên
> máy host, kiểm soát toàn bộ hệ thống chứ không chỉ một container.
>
> Lệnh `USER appuser` (Dockerfile dòng 35, 41) cắt đứt chuỗi ở bước (3)–(4):
> tiến trình `/ask` chạy với UID 10001 không có quyền ghi ngoài phạm vi được
> cấp, không cài được package hệ thống, và namespace/kernel exploit cần root
> để kích hoạt sẽ không chạy được nữa. Kẻ tấn công chiếm được shell vẫn chỉ là
> `appuser` — thiệt hại giới hạn trong quyền hạn rất hẹp đó, không lan ra
> được host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt được: gửi 10 request vào giây
> 59 của phút X — bộ đếm cố định của phút X đang ở 0..9, cả 10 đều lọt qua
> (đúng hạn mức 10/phút). Ngay giây 00 của phút X+1, bộ đếm reset về 0, nên
> gửi tiếp 10 request nữa vào giây 00–01 cũng đều hợp lệ theo đúng luật của
> bộ đếm mới. Tổng cộng 20 request lọt qua trong cửa sổ thực tế chỉ 2 giây
> (giây 59 → giây 01), dù mỗi phút riêng lẻ đều "đúng" giới hạn 10.
>
> `RateLimiter` trong code dùng ZSET với `WINDOW_SECONDS = 60` trượt liên tục
> theo `now`, không neo vào ranh giới phút đồng hồ, nên `hit_count` luôn đếm
> đúng số request trong đúng 60 giây gần nhất bất kể request đó rơi vào giây
> nào — request thứ 11 trong bất kỳ cửa sổ 60 giây nào cũng bị chặn 429, lỗ
> hổng "20 request trong 2 giây" ở trên không xảy ra được.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau ở đơn vị giới hạn: `RateLimiter` (`app/rate_limiter.py`) đếm **số
> lượng request** trong 60 giây bất kể mỗi request tốn bao nhiêu tiền;
> `CostGuard` (`app/cost_guard.py`) cộng dồn **số tiền** trong tháng bất kể
> có bao nhiêu request. Một cái chặn theo tần suất, một cái chặn theo ngân
> sách — độc lập với nhau, `main.py` gọi cả hai (`limiter.check` rồi
> `guard.check`) trước khi tốn tiền gọi LLM.
>
> Rate limit cho qua, cost guard chặn: user chỉ gửi 2 request/phút (rất dưới
> hạn mức 10/phút), nhưng mỗi câu hỏi kèm lịch sử hội thoại dài (gần
> `HISTORY_MAX_MESSAGES = 20` message) nên `tokens_in`/`tokens_out` rất lớn,
> `cost_usd` mỗi lần cao — tới request thứ 2 hoặc 3 đã cộng dồn vượt
> `monthly_budget_usd`. `limiter.check` không phản đối (mới 2-3 request/phút),
> nhưng `guard.check` ném 402.
>
> Ngược lại: user gửi câu hỏi cực ngắn, rẻ, gửi liên tục 11 request trong 60
> giây. `spent(user_id) + estimated_cost` vẫn còn xa `monthly_budget_usd` nên
> `guard.check` sẽ cho qua, nhưng `limiter.check` đã thấy `hit_count >= 10` ở
> request thứ 11 và ném 429 trước khi guard kịp được gọi tới.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện: (1) Redis mất kết nối; trên cả 3 container, `store.ping()`
> bắt đầu trả `False` vì gọi Redis timeout/lỗi (`ConversationStore.ping`
> bắt Exception và trả `False`). (2) Vì endpoint gộp giờ đóng luôn vai trò
> liveness probe và có kiểm tra Redis, cả 3 container cùng trả 503 gần như
> đồng thời — dù process bên trong hoàn toàn khỏe mạnh, chỉ Redis tạm gián
> đoạn. (3) Healthcheck của Docker/orchestrator (interval 30s, retries 3
> như trong `Dockerfile`/`docker-compose.yml`) ghi nhận đủ số lần thất bại
> liên tiếp và đánh dấu cả 3 container "unhealthy" cùng lúc. (4) Orchestrator
> restart cả 3 container gần như đồng thời — trong lúc chúng khởi động lại
> (vài giây), **không còn container nào phục vụ được request** dù nguyên
> nhân gốc chỉ là Redis chập chờn 30 giây, gây downtime hoàn toàn thay vì chỉ
> từ chối request cần Redis. (5) Khi Redis nối lại được trong lúc 30 giây đó,
> có thể một vài container đã kịp phục hồi trước khi bị kill hẳn, nhưng nhìn
> chung thiệt hại "restart storm" đã xảy ra.
>
> Đây đúng là lý do code tách `/health` (không đụng Redis, chỉ trả lời "process
> còn sống") khỏi `/ready` (được phép kiểm tra Redis, dùng để loại instance ra
> khỏi load balancer chứ không restart nó).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis như hiện tại: chạy `docker compose up --scale agent=3` rồi gọi
> `/ask` nhiều lần cùng `X-User-Id`, `history_length` tăng đều và đúng thứ tự
> (0, 1, 2, 3, ...) bất kể request đó rơi vào container nào, vì `history:<user>`
> nằm trong Redis — một nơi cả 3 container cùng đọc/ghi. Tôi kiểm tra local
> (không scale, chỉ 1 instance nhưng cùng cơ chế store): gọi `/ask` hai lần
> liên tiếp với `X-User-Id: sv-test` cho ra `history_length` lần lượt là `0`
> rồi `2` — đúng như append 2 message (user + assistant) mỗi lượt.
>
> Nếu lịch sử được lưu trong một dict Python trong RAM của từng process, thì
> mỗi container có một dict riêng, rỗng lúc khởi động. Với 3 container đứng
> sau load balancer round-robin, hai request liên tiếp của cùng một user rất
> có thể rơi vào hai container khác nhau — container thứ hai không hề biết
> user này đã hỏi gì ở container thứ nhất. `history_length` sẽ không tăng đều
> mà nhảy lung tung kiểu 0, 0, 1, 0, 1, 2, 0... tùy request rơi vào instance
> nào, và agent "quên" câu hỏi trước đó dù chỉ cách nhau một request.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp: `railway up` deploy nhầm code FastAPI của agent **đè lên service
> Redis** thay vì tạo service riêng cho agent. Nguyên nhân là Railway CLI có
> khái niệm "linked service" — lệnh chạy trên service nào đang được link vào
> thư mục hiện tại, và lúc đó CLI đang link vào `Redis` (service tôi tạo trước
> đó bằng `railway add --database redis`) chứ không phải một service `agent`
> riêng, vì tôi chưa tạo service đó trước khi chạy `railway up`.
>
> Cách tôi tìm ra nguyên nhân: ngay sau khi `railway up` chạy xong, tôi chạy
> `railway status` và thấy service `Redis` chuyển từ `Online` sang `Crashed`.
> Xem tiếp `railway logs` thì thấy lặp lại lỗi
> `/bin/sh: 1: exec: docker-entrypoint.sh: not found` — đây không phải lỗi bình
> thường của Redis (image `redis:8.2` không có vấn đề gì cả), mà là dấu hiệu
> container đang cố chạy `startCommand` trong `railway.toml` (viết cho FastAPI:
> `uvicorn app.main:app ...`) trên một image không phải Python. Từ đó suy ra
> `railway up` đã build và deploy nhầm service.
>
> Cách sửa: chạy `railway down --yes` để gỡ deployment lỗi vừa đè lên Redis,
> đưa Redis về trạng thái Offline nhưng giữ nguyên cấu hình/volume gốc; sau đó
> vào dashboard, service `Redis` hiện gợi ý sẵn nút "Deploy the image
> redis:8.2" — bấm vào đó để Redis quay lại đúng image gốc và Online trở lại.
> Để tránh lặp lại lỗi, tôi tạo hẳn một service riêng cho code bằng
> `railway add --service agent`, rồi `railway service link agent` để chắc chắn
> CLI đang trỏ đúng service `agent` (kiểm tra lại bằng `railway status`, thấy
> đúng service ID khác với Redis) trước khi chạy `railway up` lần nữa. Bài học:
> với CLI có khái niệm "service đang link", luôn `railway status` kiểm tra
> đang đứng ở service nào trước khi chạy lệnh có thể ghi đè.
