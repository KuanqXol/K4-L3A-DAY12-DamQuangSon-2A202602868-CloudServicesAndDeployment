# Checklist hoàn tất repo Cloud Services & Deployment

> Nguồn tổng hợp: `README.md`, `LAB_GUIDE.md`, `CHECKPOINTS.md`, `RUBRIC.md`,
> `RULES.md`, `SUBMISSION.md`, các file khung trong `app/` và toàn bộ test.
>
> Ký hiệu: `[ ]` chưa làm, `[x]` đã làm. Hoàn thành phần bắt buộc trước, sau đó
> mới làm mục Bonus.

## 0. Quy tắc bắt buộc và chuẩn bị repo

- [ ] Xác nhận đây là bài làm cá nhân; chỉ nộp code mình hiểu và giải thích được.
- [ ] Đổi/tạo repo theo đúng mẫu
  `K4-L3A-DAY12-<HoVaTen>-<MSSV>-CloudServicesAndDeployment`.
  - [ ] Họ tên viết liền, không dấu, viết hoa chữ cái đầu mỗi từ.
  - [ ] Dùng đúng MSSV được cấp, đúng `DAY12`, không có khoảng trắng.
  - [ ] Kiểm tra lại phần MSSV trong tên repo hiện tại; tên thư mục hiện có là
    `...-2A202602868-...`, trong khi ví dụ của đề dùng dạng `L3A...`.
- [ ] Đặt repo GitHub ở chế độ **public**.
- [ ] Giữ nguyên bộ test và `grade.py` để Lab Coach có thể chấm lại.
- [ ] Không sao chép code/bài phản ánh của học viên khác và không tạo lịch sử
  commit chung với người khác.
- [ ] Không commit `.env`, API key, token, mật khẩu, private key hoặc credential.
- [ ] Nếu secret từng bị public: thu hồi/rotate ngay và làm sạch lịch sử Git;
  chỉ xóa ở commit mới là chưa đủ.
- [ ] Commit sau từng checkpoint để lịch sử thể hiện tiến trình, không dồn thành
  một commit cuối.

## 1. CP0 — Cài đặt môi trường

- [x] Cài Python 3.11+, Git, Docker và Docker Compose.
- [ ] Chuẩn bị tài khoản GitHub và Railway hoặc Render cho CP5.
- [x] Tạo và kích hoạt virtual environment.

  ```powershell
  python -m venv .venv
  .venv\Scripts\Activate.ps1
  pip install -r requirements.txt
  ```

- [x] Copy `.env.example` thành `.env` cục bộ.
- [x] Sinh `AGENT_API_KEY` riêng bằng `secrets.token_urlsafe(32)` và đặt trong
  `.env`; không dùng giá trị mẫu và không commit file này.
- [x] Khởi động Redis bằng `docker compose up -d redis` và kiểm tra trạng thái;
  hoặc tạm dùng `REDIS_URL=fake://` khi chưa có Docker.
- [x] Chạy kiểm tra môi trường, bảo đảm pytest chạy được và không có
  `ModuleNotFoundError`/`ImportError`:

  ```powershell
  pytest tests/ -v -m "not docker"
  ```

- [x] Commit checkpoint 0.

## 2. CP1 — 12-Factor Config, Health và Logging (15 điểm)

### `app/config.py`

- [x] Khai báo đủ 6 trường trong `Settings`:
  - [x] `port: int = 8000`
  - [x] `agent_api_key: str` — bắt buộc, **không có mặc định**.
  - [x] `redis_url: str = "redis://localhost:6379/0"`
  - [x] `rate_limit_per_minute: int = 10`
  - [x] `monthly_budget_usd: float = 10.0`
  - [x] `log_level: str = "INFO"`
- [x] Bảo đảm cấu hình đọc từ environment/`.env`, thay biến môi trường thì giá
  trị thay đổi mà không sửa code.
- [x] Bảo đảm thiếu `AGENT_API_KEY` làm app fail fast khi khởi động.
- [x] Không hardcode secret trong `config.py`, `main.py` hoặc `auth.py`.

### `app/logging_utils.py`

- [x] Cài `log_event()` để tạo, in ra stdout và trả về một chuỗi JSON hợp lệ.
- [x] Mỗi log chỉ nằm trên **một dòng**, không dùng `indent`.
- [x] Log luôn có `event`, `level` viết thường và timestamp ISO-8601 UTC.
- [x] Gộp đầy đủ các trường tùy ý từ `**fields`.
- [x] Dùng `ensure_ascii=False` để giữ đúng Unicode/tiếng Việt.

### `/health` trong `app/main.py`

- [x] Bình thường trả HTTP 200 cùng `status: ok`, tên service và version.
- [x] Khi `lifecycle.shutting_down` là true, trả HTTP 503 với
  `{"status": "shutting_down"}`.
- [x] Không yêu cầu API key.
- [x] Không nhận dependency và không gọi Redis/database/dịch vụ ngoài.

### Kiểm tra CP1

- [x] Chạy `pytest tests/test_cp1.py -v` và sửa đến khi xanh toàn bộ.
- [x] Chạy app bằng Uvicorn và gọi thật `GET /health`.
- [x] Có thể giải thích: 12-Factor config, fail fast, JSON log một dòng và lý do
  liveness không kiểm tra Redis.
- [x] Commit checkpoint 1.

## 3. CP2 — Docker production-ready (15 điểm)

### `Dockerfile`

- [x] Chuyển thành multi-stage build có stage đặt tên (ví dụ `builder`) và
  stage runtime riêng.
- [x] Dùng base image gọn như `python:3.11-slim` hoặc Alpine.
- [x] Stage builder cài dependency; runtime chỉ copy kết quả cần thiết, không
  mang compiler/build tools sang image cuối.
- [x] `COPY requirements.txt` rồi `pip install` **trước** khi copy source để tận
  dụng layer cache.
- [x] Copy đủ `app/` và `utils/` vào runtime image.
- [x] Tạo user thường (ví dụ UID 10001) và dùng `USER`; container cuối không
  chạy bằng root.
- [x] Thêm `HEALTHCHECK` gọi `/health`.
- [x] Uvicorn bind `0.0.0.0`.
- [x] Đọc cổng từ `${PORT:-8000}`, không cố định cổng production.
- [x] Không chứa `AGENT_API_KEY` hoặc bất kỳ secret hardcode nào.
- [x] Build thành công và image cuối nhỏ hơn 500 MB.

### `.dockerignore`

- [x] Bổ sung tối thiểu `.env`, `__pycache__`, `.git`, `.venv`.
- [x] Không ignore nhầm `app`, `utils` hoặc `requirements.txt`.

### `docker-compose.yml`

- [x] Giữ service `redis` và bổ sung service `agent`.
- [x] `agent` build từ Dockerfile trong repo.
- [x] Map cổng `8000:8000`.
- [x] Truyền `AGENT_API_KEY: ${AGENT_API_KEY}`, không viết thẳng secret.
- [x] Đặt `REDIS_URL=redis://redis:6379/0`, không dùng localhost trong container.
- [x] Cho `agent` phụ thuộc `redis`.
- [x] Thêm healthcheck gọi `/health` cho `agent`.

### Kiểm tra CP2

- [x] Chạy kiểm tra cấu trúc nhanh:
  `pytest tests/test_cp2.py -v -m "not docker"`.
- [x] Chạy `docker build -t day12-agent:prod .` thành công.
- [x] Dùng `docker images day12-agent:prod` xác nhận image dưới 500 MB.
- [x] Chạy `docker compose up -d`, kiểm tra `docker compose ps`, log của agent
  và gọi `http://localhost:8000/health` thành công.
- [x] Chạy đầy đủ `pytest tests/test_cp2.py -v` khi Docker daemon đang bật.
- [x] Có thể giải thích: multi-stage, layer cache, network giữa container, rủi
  ro chạy root và nguy cơ secret lọt vào build context.
- [x] Commit checkpoint 2.

## 4. CP3 — API Security (20 điểm)

### `app/auth.py`

- [x] Đọc khóa đúng từ `get_settings().agent_api_key`.
- [x] Thiếu hoặc sai header `X-API-Key` trả HTTP 401 với thông báo phù hợp.
- [x] So sánh khóa bằng `secrets.compare_digest`, không dùng `==`.
- [x] Khóa đúng thì trả `X-User-Id`; nếu thiếu user ID thì trả
  `ANONYMOUS_USER`.

### `app/rate_limiter.py`

- [x] Cài sliding window 60 giây bằng Redis Sorted Set, key riêng theo user.
- [x] `hit_count()` xóa entry hết hạn bằng `zremrangebyscore` rồi đếm `zcard`.
- [x] `check()` kiểm tra quota **trước**, ghi nhận request **sau**.
- [x] Vượt giới hạn trả HTTP 429 và header `Retry-After: 60`.
- [x] Member ZSET là duy nhất cho mỗi request (timestamp + UUID).
- [x] Đặt TTL 60 giây cho key.
- [x] Hỗ trợ tham số `now` để kiểm thử cửa sổ trượt chính xác.

### `app/cost_guard.py`

- [x] Dùng key `cost:<user>:<YYYY-MM>` để tách user và tự reset theo tháng.
- [x] `spent()` trả `0.0` khi key chưa tồn tại và ép dữ liệu Redis về float.
- [x] `check()` chặn HTTP 402 khi `spent + estimated_cost > budget`.
- [x] `record()` cộng dồn bằng `incrbyfloat`, trả tổng mới và đặt TTL khoảng
  40 ngày.

### `/ask` trong `app/main.py`

- [x] Áp dụng auth dependency trước mọi xử lý khác.
- [x] Thực hiện đúng thứ tự:
  `limiter.check` → `guard.check` → đọc history → gọi mock LLM → lưu message
  user và assistant → ghi chi phí → ghi log.
- [x] Không gọi LLM nếu auth/rate limit/cost guard đã chặn request.
- [x] Response có đủ `answer`, `user_id`, `history_length`, `cost_usd` và
  `tokens.in`/`tokens.out`.
- [x] Câu hỏi rỗng bị Pydantic chặn với HTTP 422.
- [x] Log `ask_completed` có user, token vào/ra và chi phí.

### Kiểm tra CP3

- [x] Chạy `pytest tests/test_cp3.py -v` và sửa đến khi xanh toàn bộ.
- [x] Curl không key/sai key được 401; key đúng được 200.
- [x] Gọi quá hạn mức được 429; user khác có quota riêng.
- [x] Test cost guard được 402 khi vượt ngân sách và có ghi nhận chi phí sau
  request thành công.
- [x] Có thể giải thích: 401/402/429, timing attack, sliding window, member duy
  nhất và khác biệt giữa rate limit với budget limit.
- [x] Commit checkpoint 3.

## 5. CP4 — Scaling và Reliability (20 điểm)

### `app/store.py`

- [x] Lưu toàn bộ history trong Redis, không dùng dict/global state trong process.
- [x] `ping()` trả true khi Redis sống; bắt mọi exception và trả false khi lỗi.
- [x] `append()` lưu JSON Unicode gồm `role` và `content` vào Redis List.
- [x] `ltrim(key, -HISTORY_MAX_MESSAGES, -1)` để chỉ giữ 20 message mới nhất.
- [x] Đặt TTL 7 ngày cho history.
- [x] `get_history()` đọc theo thứ tự cũ đến mới, parse JSON và trả list rỗng
  khi chưa có dữ liệu.
- [x] Mỗi user có history riêng; history được dùng lại giữa các request.

### `/ready` và trạng thái shutdown

- [x] Redis sống: `/ready` trả 200 với `status: ready`, `redis: true`.
- [x] Redis lỗi: `/ready` trả 503 với `status: not ready`, `redis: false`.
- [x] Đang shutdown: cả `/health` và `/ready` trả 503 `shutting_down`.
- [x] Giữ `/health` và `/ready` tách biệt đúng vai trò liveness/readiness.

### `app/lifecycle.py`

- [x] `request_shutdown()` chỉ bật cờ `shutting_down` và gọi lại handler cũ
  nếu handler đó callable.
- [x] `install()` lưu handler cũ rồi đăng ký handler mới cho cả SIGTERM và
  SIGINT.
- [x] Không làm I/O hoặc công việc nặng trong signal handler.

### Kiểm tra CP4

- [x] Chạy `pytest tests/test_cp4.py -v` và sửa đến khi xanh toàn bộ.
- [x] Chạy nhiều instance bằng Docker Compose và xác nhận history dùng chung:
  `docker compose up -d --scale agent=3`.
- [x] Gọi `/ask` nhiều lần cùng `X-User-Id`; `history_length` phải tăng nhất
  quán dù request có thể vào instance khác nhau.
- [x] Có thể giải thích: stateless service, giới hạn/TTL history, khác biệt giữa
  liveness và readiness, và cách graceful shutdown tránh rớt request.
- [x] Commit checkpoint 4.

## 6. CP5 — Deploy cloud (15 điểm)

### Deploy service

- [ ] Chọn Railway, Render hoặc Cloud Run (Railway là đường ngắn nhất theo lab).
- [ ] Push đầy đủ code lên GitHub trước khi kết nối platform.
- [ ] Deploy từ Dockerfile; app phải bind `0.0.0.0` và dùng `$PORT` của platform.
- [ ] Tạo/kết nối Redis thật trên cloud; không dùng `fake://` khi deploy.
- [ ] Đặt các biến trên dashboard/secret store, không đặt secret trong repo:
  - [ ] `AGENT_API_KEY`
  - [ ] `REDIS_URL`
  - [ ] `RATE_LIMIT_PER_MINUTE=10`
  - [ ] `MONTHLY_BUDGET_USD=10.0`
  - [ ] `LOG_LEVEL=INFO`
  - [ ] `PORT` để platform tự cấp, không ghi đè nếu không cần.
- [ ] Kiểm tra build log và runtime log; sửa mọi lỗi build/start/health check.
- [ ] Tạo domain HTTPS công khai.

### Kiểm tra bản deploy thật

- [ ] `GET <URL>/health` trả 200 và `status: ok`.
- [ ] `GET <URL>/ready` trả 200 và `status: ready`, chứng minh Redis đã nối.
- [ ] `POST <URL>/ask` không key trả 401.
- [ ] `POST <URL>/ask` có key thật trả 200 và có câu trả lời.
- [ ] Gọi 15 lần để xác nhận các request cuối trả 429.
- [ ] Có thể đặt `DEPLOY_API_KEY` trong `.env` cục bộ để test CP5 kiểm tra thêm
  đường xác thực; tuyệt đối không commit giá trị này.

### Hoàn thiện bằng chứng

- [x] Điền `DEPLOYMENT.md`, xóa toàn bộ placeholder `(điền...)`, `TODO`, URL mẫu.
- [x] Điền họ tên, MSSV và link repo đúng.
- [x] Điền Public URL HTTPS, platform và ngày deploy.
- [x] Liệt kê **tên** và nguồn các biến môi trường; không ghi giá trị secret.
- [x] Dán output thực của các lệnh kiểm tra vào `DEPLOYMENT.md`.
- [x] Thêm `screenshots/dashboard.png` — dashboard của service.
- [x] Thêm `screenshots/health.png` — kết quả gọi `/health`.
- [x] Chạy `pytest tests/test_cp5.py -v` và sửa đến khi đạt.
- [x] Có thể giải thích `$PORT`, secret store, build/runtime log, health/readiness
  và lỗi thực tế đã gặp khi deploy.
- [x] Commit checkpoint 5.

### Phương án dự phòng nếu không thể dùng cloud

- [x] Chỉ dùng khi thực sự không deploy được; hiểu rằng CP5 tối đa 9/15 điểm.
- [x] Đặt `LOCAL_FALLBACK=true` trong `.env` cục bộ.
- [x] Chạy `docker compose up -d` và xác nhận stack healthy.
- [x] Chụp ít nhất một ảnh trong `screenshots/` có `docker compose ps` và kết
  quả gọi API.
- [x] Ghi rõ lý do không deploy được trong `DEPLOYMENT.md`.
- [x] Chạy lại `pytest tests/test_cp5.py -v` ở fallback mode.

## 7. Hoàn thành `exercises.md` (15 điểm)

- [x] Điền họ tên và mã học viên.
- [x] Trả lời đủ 10 câu bằng lời của chính mình, dựa trên output/quan sát thật:
  - [x] Câu 1: tình huống fail fast cứu hệ thống khi thiếu API key.
  - [x] Câu 2: một dòng log JSON thật và hai lợi ích so với `print` chung chung.
  - [x] Câu 3: số MB thật của image single-stage và multi-stage, giải thích phần
    chênh lệch.
  - [x] Câu 4: layer nào cache/tái build sau khi sửa source và tác hại của
    `COPY . .` trước `pip install`.
  - [x] Câu 5: chuỗi rủi ro từ lỗ hổng Python đến quyền root host và vai trò
    của `USER`.
  - [x] Câu 6: số request tối đa trong 2 giây với fixed window 10/phút và cách
    tạo burst đó.
  - [x] Câu 7: ví dụ rate limit cho qua nhưng cost guard chặn, và ngược lại.
  - [x] Câu 8: chuỗi sự kiện khi gộp health/ready và Redis mất 30 giây trong
    cụm 3 container.
  - [x] Câu 9: quan sát `history_length` khi scale 3 instance và so sánh với
    lưu history trong dict.
  - [x] Câu 10: một lỗi deploy thật, thông báo lỗi, cách tìm nguyên nhân và cách sửa.
- [x] Không còn dòng `> *Câu trả lời của bạn*` hoặc câu trả lời chung chung.
- [x] Có thể giải thích miệng toàn bộ nội dung đã viết.

## 8. Kiểm tra bảo mật và chất lượng trước khi nộp

- [x] Tìm và xử lý toàn bộ `NotImplementedError` trong `app/`.
- [x] Không còn TODO bắt buộc hoặc placeholder trong code/tài liệu nộp.
- [x] Chạy toàn bộ test:

  ```powershell
  pytest tests/ -v
  ```

- [x] Ghi nhận rõ test nào pass/fail/skip và nguyên nhân; test bị skip vì thiếu
  Docker không tự động được xem là đã đạt.
- [x] Chạy chấm điểm bắt buộc: `python grade.py --no-bonus`.
- [x] Chạy chấm điểm đầy đủ: `python grade.py`.
- [x] Mục tiêu ít nhất 75/100; ưu tiên làm xanh toàn bộ phần bắt buộc.
- [x] Kiểm tra Git không theo dõi `.env` hoặc key/private key:

  ```powershell
  git status --short
  git ls-files | Select-String -Pattern '(^|/)\.env$|\.(pem|key)$'
  ```

- [x] Kết quả kiểm tra chỉ có thể chứa `.env.example`, không có `.env` thật.
- [x] Quét repo/lịch sử để chắc chắn không có API key, token hay mật khẩu thật.
- [x] Kiểm tra `DEPLOYMENT.md` không chứa secret và không còn placeholder.
- [x] Kiểm tra đủ source `app/`, `utils/`, Dockerfile, Compose, `.dockerignore`,
  file deploy, test, grade script, exercises và screenshots.
- [x] Đọc lại code và chuẩn bị giải thích mọi phần khi Lab Coach hỏi.

## 9. Nộp bài

- [ ] Commit các thay đổi cuối với thông điệp rõ ràng.
- [ ] Push toàn bộ commit lên nhánh chính của GitHub.
- [ ] Mở repo public từ một phiên đăng xuất để xác nhận Lab Coach truy cập được.
- [ ] Xác nhận tên repo chính xác lần cuối; sai tên bị trừ 5 điểm.
- [ ] Nộp **link repository GitHub** lên Codelab/LMS theo hướng dẫn lớp.

## 10. Bonus — GitHub Actions CI/CD (không bắt buộc, tối đa +10)

> Chỉ bắt đầu sau khi CP1–CP5 và phần bắt buộc đã ổn. Nginx/load balancing là
> mở rộng kiến thức, không có điểm bonus riêng.

- [x] Tạo `.github/workflows/ci.yml` (hoặc `.yaml`).
- [x] Kích hoạt workflow khi push và pull request vào `main`.
- [x] Tạo job test trên runner sạch:
  - [x] Checkout code bằng action đã ghim phiên bản, ví dụ `actions/checkout@v4`.
  - [x] Setup Python bằng action đã ghim phiên bản.
  - [x] Cài `requirements.txt`.
  - [x] Truyền `AGENT_API_KEY=ci-dummy` và `REDIS_URL=fake://` qua `env`.
  - [x] Chạy pytest nhưng loại `test_cp5.py` và `test_bonus_cicd.py` khỏi CI.
- [x] Tạo job/bước build Docker image trên GitHub runner.
- [x] Tạo job deploy thật.
- [x] Dùng `needs` để deploy chỉ chạy sau khi cả test và build xanh.
- [x] Dùng `if` để deploy chỉ chạy khi push vào `main`, không deploy từ pull request.
- [x] Lưu Railway token/Render deploy hook trong GitHub Actions Secrets và tham
  chiếu bằng `${{ secrets.* }}`; không hardcode token.
- [x] Lưu giá trị không bí mật như public URL trong GitHub Variables.
- [x] Ghim version/SHA cho mọi action; không dùng `@main`, `@master`, `@latest`.
- [x] Thêm smoke test sau deploy gọi `${{ vars.PUBLIC_URL }}/health` và làm job
  fail nếu HTTP không phải 2xx.
- [x] Thêm badge workflow đúng repo/file vào đầu `README.md`.
- [ ] Push workflow, xem tab Actions và sửa đến khi lần chạy mới nhất xanh.
- [ ] Xác nhận badge tải được và hiển thị `passing` trên repo public.
- [ ] Chạy `pytest tests/test_bonus_cicd.py -v` đến khi xanh toàn bộ.

## 11. Mở rộng tùy chọn, không tính bonus

- [ ] Thêm service Nginx dùng `nginx/nginx.conf` làm load balancer.
- [ ] Chạy nhiều agent bằng `docker compose up -d --scale agent=3` và gọi qua
  Nginx/cổng 80 để quan sát cân bằng tải.

## Điều kiện hoàn tất cuối cùng

- [ ] `pytest tests/ -v` đã được chạy trong môi trường có Docker và không còn
  lỗi bắt buộc.
- [ ] `python grade.py` cho kết quả mục tiêu.
- [ ] Service cloud HTTPS đang hoạt động; `/health`, `/ready`, auth, rate limit
  và Redis đều được kiểm chứng.
- [ ] `exercises.md`, `DEPLOYMENT.md` và screenshots đầy đủ, dựa trên kết quả thật.
- [ ] Repo public đúng tên, không có secret, có nhiều commit theo tiến trình và
  đã nộp đúng link.
