# Thông Tin Deploy — Checkpoint 5

> Mã nguồn đã triển khai trên Railway. Các kết quả gọi API bên dưới được kiểm tra
> qua URL công khai. `pytest tests/test_cp5.py` đọc file này.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đoàn Quang Thắng |
| Mã học viên | 2A202602395 |
| Repo | https://github.com/conanWinner/K4-L3B-DAY12-DoanQuangThang-2A202602395-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-935d.up.railway.app |
| Platform | Railway |
| Railway project | [day12-agent-lab](https://railway.com/project/d3f0be7a-e95f-454e-8a75-5c4d9f4f2cc0), môi trường `production` |
| Ngày deploy | 29/09/2026 |
| Trạng thái | Redis và `day12-agent` đều `SUCCESS`, tiến trình `RUNNING`; URL công khai hoạt động |
| Bản triển khai | `714019a5-6e9f-4439-a35d-4f84eb9bb448` |
| Kiểm tra sức khỏe | Railway dùng `/ready`; bản triển khai dùng `Dockerfile` |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | Tự động | Railway cấp cổng cho service; ứng dụng đọc `$PORT` |
| `AGENT_API_KEY` | ✅ | Khóa riêng của cloud, đặt qua Railway CLI stdin; giá trị không nằm trong repo |
| `REDIS_URL` | ✅ | Biến tham chiếu `${{Redis.REDIS_URL}}` của cùng project |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Chạy trong thư mục gốc của repo. `DEPLOY_API_KEY` trong `.env` là khóa riêng của
dịch vụ Railway; lệnh `source` nạp nó vào terminal nhưng không in ra màn hình.

```bash
set -a
source .env
set +a
URL="https://day12-agent-production-935d.up.railway.app"

# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i "$URL/health"

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i "$URL/ready"

# 3. Không có API key — mong đợi 401
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $DEPLOY_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — dùng user riêng để không lẫn bộ đếm với phép thử trên
RATE_USER="cp5-rate-$(date +%s)"
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST "$URL/ask" \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $DEPLOY_API_KEY" \
    -H "X-User-Id: $RATE_USER" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Đã kiểm tra qua URL công khai ngày 29/09/2026:

```text
GET  /health                 200  {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET  /ready                  200  {"status":"ready","redis":true}
POST /ask (không có key)     401  {"detail":"invalid or missing API key"}
POST /ask (có key cloud)    200  {"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"cp5-doc-check","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}
POST /ask x15, cùng user     200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

Các phản hồi đơn lẻ được kiểm tra lại ngày 29/09/2026; dãy 15 lượt được ghi
từ lần thử giới hạn tốc độ trước đó. Header động được lược khỏi bản ghi. Khóa
API không xuất hiện trong output.

## Kiểm Tra Tự Động

Ngày 29/09/2026, chạy `pytest` cho CP1–CP5 với bộ lọc `-m "not docker"`:

```text
77 passed, 4 skipped, 2 deselected
```

Riêng `tests/test_cp5.py`: `9 passed, 4 skipped`. Bốn bài skipped chỉ dùng
cho phương án dự phòng chạy cục bộ. Lệnh `grade.py --no-bonus` với cùng bộ lọc
cho kết quả tự chấm `100/100` điểm bắt buộc; hai bài dựng lại Docker image
không chạy trong lượt kiểm tra này. Chất lượng phần trả lời trong `exercises.md`
vẫn do giảng viên đánh giá.

## Ảnh Chụp Màn Hình

Đã kiểm tra ba ảnh trong thư mục `screenshots/`:

- [screenshots/Dashboard_1.png](screenshots/Dashboard_1.png) — Railway dashboard cho thấy `day12-agent` và Redis đều Online
- [screenshots/Dashboard_2.png](screenshots/Dashboard_2.png) — bản triển khai đang Active, log cho thấy `/health`, `/ready` trả 200 và `/ask` thiếu khóa trả 401
- [screenshots/Health.png](screenshots/Health.png) — trình duyệt gọi URL công khai `/health`, nhận `status: ok`

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

Không áp dụng: đã chọn Railway.
