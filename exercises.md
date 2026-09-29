# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời chi tiết của bạn vào từng câu bên dưới.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đoàn Quang Thắng  Mã học viên: 2A202602395

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> **Tình huống:** Khi deploy ứng dụng lên môi trường Production trên Cloud (hoặc tạo một môi trường staging mới), lập trình viên quên khai báo biến môi trường `AGENT_API_KEY` trong dashboard.
> - **Nếu có giá trị mặc định `"changeme"`:** Ứng dụng vẫn khởi động thành công và báo trạng thái healthy. Bất kỳ ai hoặc các bot quét lỗ hổng trên Internet có thể gọi vào hệ thống với key mặc định `"changeme"`, làm rò rỉ dữ liệu hoặc tiêu cạn ngân sách OpenAI/Anthropic/Gemini của bạn. Nguy hiểm hơn, dev tưởng ứng dụng đã chạy tốt mà không hề hay biết lỗi cho đến khi nhận hóa đơn LLM khổng lồ.
> - **Khi không có mặc định (Fail Fast):** Pydantic ném ngay ngoại lệ `ValidationError` lúc nạp cấu hình và tiến trình app crash ngay lập tức. Deployment trên cloud lập tức báo lỗi đỏ, container không được cấp quyền nhận traffic, buộc lập trình viên phải mở dashboard và cấu hình ngay secret key trước khi ứng dụng có thể phục vụ người dùng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> **Dòng log JSON thu được:**
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T11:10:30+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00012}
> ```
>
> **Hai việc làm được với log JSON mà `print()` tự do không thể làm:**
> 1. **Lọc, tổng hợp và phân tích định lượng tự động (Log Aggregation & Metrics):** Các hệ thống thu thập log tập trung (như Datadog, Grafana Loki, ELK, CloudWatch) có thể parse trường JSON tự động để thực hiện các câu truy vấn phức tạp như: *"Tổng chi phí (cost_usd) của user `sv-test` trong ngày là bao nhiêu?"*, *"Lượng token tiêu thụ trung bình mỗi request theo từng giờ"*. Chuỗi text không cấu trúc của `print()` không thể bóc tách các trường số liệu này một cách tin cậy và có hiệu năng cao ở quy mô hàng triệu log.
> 2. **Thiết lập hệ thống cảnh báo tự động theo ngưỡng (Automated Alerting):** Có thể cài đặt trực tiếp quy tắc cảnh báo (Alert Rule) trên hệ thống giám sát dựa trên giá trị số học: ví dụ tự động kích hoạt cảnh báo gửi tới Slack/PagerDuty khi `cost_usd > 0.05` ở một request đơn lẻ (phát hiện prompt injection tiêu tốn token bất thường) hoặc khi số lượng log `level: "error"` tăng đột biến trong 5 phút.

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
| 1 stage (bản đầu) | ~1050 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> **Giải thích phần chênh lệch (~780 MB):**
> 1. **Base image:** Bản 1-stage ban đầu dùng `python:3.11` đầy đủ (dựa trên Debian standard), chứa sẵn hàng loạt công cụ phát triển phần mềm, trình biên dịch `gcc`, `g++`, các file header `C/C++ dev headers`, package managers và utilities hệ thống không bao giờ dùng lúc chạy app. Bản multi-stage chuyển sang dùng `python:3.11-slim` đã lược bỏ toàn bộ các gói thừa thãi này.
> 2. **Tách biệt môi trường build và runtime:** Bản multi-stage sử dụng stage đầu (`builder`) để chạy `pip install` và biên dịch các thư viện Python, sau đó ở stage cuối (`runtime`) chỉ copy kết quả thư viện đã cài đặt sang (`COPY --from=builder /install /usr/local`). Nhờ vậy, toàn bộ pip wheel cache, temporary build artifacts, và các công cụ hỗ trợ cài đặt đều bị loại bỏ hoàn toàn khỏi image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> **Với Dockerfile tối ưu hiện tại:**
> - Các layer được tái sử dụng từ cache (`CACHED`): Toàn bộ stage `builder` (bao gồm `WORKDIR /build`, `COPY requirements.txt .`, `RUN pip install...`) vì file `requirements.txt` không hề thay đổi. Ở stage `runtime`, layer base và lệnh cài đặt môi trường như `useradd` cũng được lấy từ cache.
> - Các layer phải chạy lại: Chỉ bắt đầu từ layer `COPY app ./app` trở đi (do nội dung thư mục `app` đã thay đổi checksum) cùng với các chỉ thị cấu hình tiếp sau (`COPY utils`, `USER`, `EXPOSE`, `CMD`). Thời gian build chỉ mất khoảng 1-2 giây.
>
> **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
> Docker sử dụng cơ chế cache theo từng layer tuần tự: khi bất kỳ một file nào trong thư mục bị sửa (dù chỉ là 1 ký tự trong `main.py`), layer `COPY . .` sẽ bị thay đổi checksum làm vô hiệu hóa toàn bộ cache từ layer đó trở xuống. Khi đó, Docker bắt buộc phải thực thi lại lệnh `RUN pip install -r requirements.txt`, tải lại và cài đặt lại toàn bộ thư viện từ Internet mỗi lần sửa code, khiến thời gian build tăng từ 2 giây lên vài phút mỗi lần.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> **Chuỗi sự kiện tấn công khi chạy root:**
> 1. **Khai thác ứng dụng (Application Exploitation):** Kẻ tấn công tìm thấy một lỗ hổng trong code Python (ví dụ: Remote Code Execution qua Command Injection, deserialize không an toàn với `pickle`, hoặc file upload tùy ý).
> 2. **Chiếm quyền root trong container (Container Root Execution):** Khi mã độc được thực thi, tiến trình kế thừa quyền hạn của tiến trình cha (FastAPI/Uvicorn). Do container chạy mặc định bằng root (UID 0), kẻ tấn công trở thành root bên trong container, toàn quyền đọc/ghi file hệ thống, mở cổng mạng và can thiệp tiến trình.
> 3. **Thoát khỏi container chiếm quyền máy Host (Container Escape):** Vì nhân Linux (kernel) được dùng chung giữa container và máy host, và mặc định UID 0 trong container map thẳng tới UID 0 trên host. Kẻ tấn công có thể lợi dụng quyền root để khai thác các lỗ hổng nhân Linux (kernel exploits), hoặc truy cập các volume/socket nhạy cảm được mount vào (như `/var/run/docker.sock`, `/proc`, `/sys`) để thoát ra ngoài container và chiếm quyền điều khiển root máy chủ host.
>
> **Lệnh `USER appuser` cắt đứt chuỗi ở đâu:**
> Lệnh `USER appuser` chuyển tiến trình chạy sang một tài khoản không có quyền quản trị (non-root UID 10001, không có quyền sudo). Khi kẻ tấn công khai thác được lỗ hổng RCE, mã độc chỉ chạy dưới quyền `appuser`: không thể ghi đè các file binary hệ thống, không thể truy cập socket của Docker daemon, và không có đủ capabilities để thực hiện các kỹ thuật container escape, qua đó chặn đứng hoàn toàn việc leo thang đặc quyền lên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **Số request tối đa trong 2 giây liên tiếp:** **20 request** (gấp 2 lần hạn mức).
>
> **Cách đạt được:**
> Với thuật toán fixed-window đếm theo phút đồng hồ:
> 1. Tại thời điểm `10:00:59` (giây 59 của phút 10), người dùng gửi dồn dập **10 request**. Hệ thống kiểm tra thấy phút 10 mới có 10 request (đạt đúng giới hạn 10) nên cho phép tất cả 10 request đi qua.
> 2. Đúng 1 giây sau, lúc `10:01:00`, đồng hồ bước sang phút mới và bộ đếm lập tức reset về 0. Người dùng gửi tiếp ngay lập tức **10 request** nữa. Hệ thống ghi nhận đây là phút 11 và tiếp tục cho phép cả 10 request đi qua.
>
> **Hậu quả:** Trong khoảng thời gian chỉ vỏn vẹn 2 giây (từ `10:00:59` đến `10:01:00`), hệ thống đã phải gánh tới 20 request, gây ra hiện tượng spike traffic làm quá tải server hoặc cạn kiệt tài nguyên downstream API.
>
> Ngược lại, thuật toán **Sliding Window (cửa sổ trượt)** tính toán chính xác tổng số request trong 60 giây gần nhất tính từ thời điểm hiện tại (`[now - 60s, now]`), do đó tại bất kỳ thời điểm nào người dùng cũng không thể gửi quá 10 request trong bất kỳ khoảng thời gian 60 giây liên tục nào.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Điểm khác biệt cốt lõi:**
> - **Rate Limit:** Kiểm soát **tốc độ / tần suất (Velocity/Concurrency)** của các request trong một khoảng thời gian ngắn (ví dụ: tối đa 10 request/phút). Mục đích chính là bảo vệ tài nguyên hạ tầng server (CPU, RAM, Network I/O) khỏi bị quá tải, spam hoặc tấn công DoS. Nó không quan tâm nội dung request to hay nhỏ, tốn bao nhiêu chi phí.
> - **Cost Guard:** Kiểm soát **tổng ngân sách tài chính tích lũy (Budget/Financial Cap)** trong một chu kỳ dài (ví dụ: tối đa $10.0/tháng). Mục đích là bảo vệ túi tiền của doanh nghiệp/người vận hành trước hóa đơn dịch vụ LLM bên ngoài (OpenAI, Anthropic, Gemini).
>
> **Tình huống 1: Rate Limit cho qua nhưng Cost Guard chặn:**
> - Người dùng gửi request đầu tiên trong vòng 10 phút (tần suất cực thấp, hoàn toàn thỏa mãn giới hạn 10 request/phút của Rate Limiter).
> - Tuy nhiên, tài khoản của người dùng này đã tích lũy chi tiêu trước đó đạt $9.995 trên hạn mức $10.0/tháng. Khi request này gửi một prompt rất dài với tài liệu đính kèm (ước tính tiêu tốn $0.05), Cost Guard kiểm tra thấy tổng chi phí vượt quá $10.0 nên chặn ngay và trả về mã lỗi `402 Payment Required`.
>
> **Tình huống 2: Cost Guard cho qua nhưng Rate Limit chặn:**
> - Đầu tháng, người dùng mới bắt đầu chu kỳ và chưa tiêu tốn đồng nào (ngân sách còn nguyên $10.0).
> - Người dùng chạy một script gửi liên tiếp 15 request ngắn (mỗi request chỉ là câu chào đơn giản, chi phí chỉ $0.0001 mỗi request, tổng cộng 15 request chỉ tốn $0.0015).
> - Chi phí $0.0015 hoàn toàn nằm trong ngân sách $10.0 của Cost Guard, nhưng do gửi 15 request trong vòng vài giây, Rate Limiter sẽ phát hiện vượt quá 10 request/phút và chặn từ request thứ 11 với mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> **Thứ tự sự kiện xảy ra khi gộp `/health` và `/ready` kiểm tra Redis:**
> 1. **Mất kết nối:** Redis gặp trục trặc mạng hoặc khởi động lại tạm thời trong vòng 30 giây.
> 2. **Healthcheck đồng loạt thất bại:** Cả 3 container agent đều kiểm tra Redis trong endpoint `/health`, phát hiện không ping được và đồng loạt trả về lỗi (HTTP 503 hoặc fail probe).
> 3. **Orchestrator kích hoạt cơ chế tự phục hồi sai lầm (Liveness action):** Bộ điều phối (Docker Compose, Kubernetes, Railway, ECS,...) xem `/health` là Liveness Probe để kiểm tra xem tiến trình container có bị dead-lock hay crash không. Nhận thấy `/health` fail ở cả 3 container, Orchestrator kết luận cả cụm ứng dụng đã hỏng và tiến hành **kill và khởi động lại (restart) toàn bộ 3 container**.
> 4. **Vòng lặp khởi động vô vọng (CrashLoop):** Các container mới khởi động lên, kiểm tra Redis vẫn đang trong 30 giây mất kết nối → lại báo unhealthy → lại bị restart liên tục.
> 5. **Hậu quả thảm họa (Cascading Failure):** Khi Redis hoàn tất khởi động sau 30 giây, lẽ ra hệ thống đã có thể phục vụ bình thường, nhưng lúc này toàn bộ 3 container agent đều đang trong trạng thái restarting hoặc khởi động dở dang. Một sự cố mạng tạm thời nhỏ của Redis đã biến thành downtime kéo dài của toàn bộ ứng dụng người dùng.
>
> *(Thiết kế đúng: `/health` (Liveness) chỉ kiểm tra process app còn chạy không để không restart bừa bãi; `/ready` (Readiness) kiểm tra Redis để Load Balancer tạm thời cô lập traffic không gửi vào cho đến khi Redis kết nối lại).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> **Sự khác biệt về `history_length` giữa Stateless (Redis) và Stateful (Dict in-memory):**
>
> - **Khi lưu trên Redis tập trung (Stateless Process):**
>   Lịch sử hội thoại được lưu tại Redis key `history:<user_id>`. Bất kể Load Balancer phân phối request tới container agent 1, 2 hay 3, container đó đều đọc và ghi vào cùng một Redis store. Người dùng quan sát thấy `history_length` tăng đều đặn và liên tục sau mỗi lượt hỏi đáp: `0 -> 2 -> 4 -> 6 -> 8...`. Trải nghiệm hội thoại diễn ra liền mạch.
>
> - **Nếu lưu trong dict Python trong RAM của từng container:**
>   Do RAM của 3 tiến trình container hoàn toàn độc lập và cách ly với nhau:
>   - Request 1 vào Container A: `history_length = 0` (A lưu lượt 1)
>   - Request 2 vào Container B: `history_length = 0` (B chưa có lịch sử của user này)
>   - Request 3 vào Container A: `history_length = 2` (A tìm thấy lượt 1)
>   - Request 4 vào Container C: `history_length = 0` (C hoàn toàn chưa có dữ liệu)
>   - Request 5 vào Container B: `history_length = 2` (B chỉ có lượt 2)
>   
>   Con số `history_length` sẽ nhảy lộn xộn và không tăng tuyến tính. Người dùng sẽ gặp hiện tượng agent "mất trí nhớ ngẫu nhiên" (lúc nhớ lúc quên câu vừa hỏi) tùy thuộc vào việc request rơi ngẫu nhiên vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi gặp phải: Endpoint `/ready` trả về HTTP 503 và ứng dụng không kết nối được Redis trên Railway.**
>
> - **Thông báo lỗi / Triệu chứng:**
>   Khi kiểm tra bản deploy bằng lệnh `curl -i https://day12-agent-production-935d.up.railway.app/ready`, server trả về mã lỗi:
>   ```http
>   HTTP/2 503
>   {"status":"not ready","redis":false}
>   ```
>   Lệnh `pytest tests/test_cp5.py` báo fail tại test `test_ready_tra_ve_200`.
>
> - **Cách tìm ra nguyên nhân:**
>   1. Kiểm tra log của service trên cloud bằng lệnh `railway logs --service day12-agent`: nhận thấy log ghi nhận không thể kết nối tới hostname Redis mặc định `redis://localhost:6379/0`.
>   2. Kiểm tra phần **Variables** của service `day12-agent` trên Railway dashboard: nhận thấy biến `REDIS_URL` chưa được khai báo hoặc đang trỏ về `localhost` (bên trong container cloud, `localhost` trỏ vào chính container app chứ không phải service Redis riêng biệt vừa tạo).
>
> - **Cách sửa:**
>   Vào Dashboard của Railway, chọn service `day12-agent` -> chuyển tới tab **Variables**, thêm biến môi trường `REDIS_URL` sử dụng biến tham chiếu nội bộ của Railway: `${{Redis.REDIS_URL}}` (với `Redis` là tên service cơ sở dữ liệu Redis cùng project). Railway sẽ tự động liên kết URL kết nối private nội bộ có mật khẩu xác thực giữa hai service. Sau khi biến được cập nhật, service tự động redeploy và endpoint `/ready` trả về mã `200 OK` với `{"status":"ready","redis":true}`.
