# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
> Cách trả lời: điền câu trả lời bên dưới từng câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đàm Quang Sơn  Mã học viên: 2A202602868

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên môi trường cloud (như Railway hoặc Render), người quản trị có thể vô tình quên thiết lập biến môi trường `AGENT_API_KEY` trong bảng điều khiển cấu hình. Nếu cấu hình để giá trị mặc định là `"changeme"`, ứng dụng vẫn sẽ khởi động thành công và mở cổng ra mạng Internet công khai. Các bot tự động dò quét lỗ hổng sẽ thử các API key mặc định phổ biến như `"changeme"`, truy cập thành công vào endpoint `/ask` và thực hiện liên tục các truy vấn LLM, khiến tài khoản của bạn bị trừ sạch tiền hoặc phải chịu hóa đơn hàng nghìn USD. Ngược lại, khi không có giá trị mặc định, `pydantic-settings` sẽ ném ngoại lệ `ValidationError` ngay lúc nạp cấu hình khi khởi động (fail-fast), container lập tức dừng và báo lỗi deploy rõ ràng trong log, buộc nhà phát triển phải bổ sung khóa bảo mật hợp lệ trước khi dịch vụ kịp phục vụ bất kỳ traffic nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:45:00.123456+00:00", "user_id": "sv01", "tokens_in": 15, "tokens_out": 32, "cost_usd": 0.00014}
```

Hai việc làm được với log JSON cấu trúc mà `print("đã trả lời xong")` không thể làm được:
1. **Truy vấn và tổng hợp số liệu tự động (Structured Querying & Analytics)**: Các hệ thống phân tích log tập trung (như Datadog, Grafana Loki, AWS CloudWatch) có thể parse trực tiếp các trường dữ liệu để tính toán thống kê theo thời gian thực, ví dụ: chạy truy vấn `sum(cost_usd) by (user_id)` để xác định người dùng tiêu tốn nhiều chi phí nhất trong ngày hoặc đếm tổng số token vào/ra.
2. **Thiết lập cảnh báo tự động theo ngưỡng (Automated Alerting)**: Có thể dễ dàng cấu hình quy tắc cảnh báo tự động kích hoạt khi xuất hiện các giá trị bất thường, ví dụ: gửi thông báo qua Slack/PagerDuty khi `cost_usd > 0.05` trên một request đơn lẻ hoặc khi tỷ lệ log `level: "error"` tăng đột biến. Với dòng `print` thông thường, hệ thống không thể bóc tách số liệu một cách tin cậy.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Số đo `271 MB` được lấy lại bằng `docker images day12-agent:cp2-test`. Phần
dung lượng chênh lệch khoảng 749 MB bao gồm:
1. **Trình biên dịch và công cụ build (Build Toolchain)**: Image Debian/Python đầy đủ chứa `gcc`, `g++`, `make`, `binutils` và các thư viện phát triển C/C++ cần thiết để biên dịch mã nguồn nhưng hoàn toàn dư thừa lúc chạy ứng dụng.
2. **Headers và tệp phát triển (`*-dev` packages)**: Các tệp header `.h` dùng để build các extension C của Python.
3. **Bộ nhớ đệm gói (Package Caches & Docs)**: Bộ nhớ tạm của `apt` (`/var/lib/apt/lists/*`), cache tải về của `pip` (`~/.cache/pip`), các trang hướng dẫn man-pages và tài liệu tài nguyên hệ thống.
Trong multi-stage build, toàn bộ môi trường biên dịch nặng nề chỉ nằm trong stage `builder` và bị hủy bỏ; stage `runtime` sử dụng base image `python:3.11-slim` chỉ chứa các thư viện tối thiểu và tệp binary đã được cài đặt vào `/usr/local`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại:
  - Các layer được dùng lại từ cache (CACHE HIT): `COPY requirements.txt .`, `RUN pip install ...` ở stage builder, cùng với base image và `COPY --from=builder /install /usr/local` ở stage runtime.
  - Các layer phải chạy lại (CACHE MISS): từ lệnh `COPY app ./app` trở đi trong stage runtime. Docker phải đánh giá lại các lệnh sau đó; các bước chỉ chứa metadata có thể hoàn thành nhanh, nhưng tôi không gán một thời gian cố định vì còn phụ thuộc cache và máy build.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  Mỗi khi thay đổi dù chỉ một ký tự trong `app/main.py`, cache của lệnh `COPY . .` lập tức bị mất hiệu lực (invalidated). Kéo theo đó, toàn bộ các layer phía sau nó—bao gồm cả lệnh tốn thời gian nhất là `RUN pip install`—đều bị hủy cache và buộc phải tải lại toàn bộ các gói thư viện từ Internet và cài đặt lại từ đầu, làm tăng thời gian build mỗi lần chỉnh sửa mã từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện leo thang đặc quyền:
  1. Ứng dụng Python tồn tại lỗ hổng bảo mật (ví dụ: Remote Code Execution qua lỗ hổng nạp module không an toàn, command injection, hoặc thư viện bên thứ ba chứa CVE).
  2. Kẻ tấn công gửi payload khai thác thành công và chiếm quyền thực thi lệnh shell tùy ý trong container.
  3. Vì container không chỉ định người dùng, tiến trình thực thi mang quyền `root` (UID 0) bên trong container.
  4. Mặc định nhân Linux chia sẻ chung kernel giữa host và container. Nếu kẻ tấn công có quyền root trong container, chúng có thể khai thác các lỗ hổng container breakout (lỗ hổng nhân Linux, cgroups, lỗ hổng `runc` / container engine, hoặc ghi đè vào các mount volume chia sẻ như Docker socket `/var/run/docker.sock`) để thoát khỏi container và chiếm trọn quyền kiểm soát máy chủ host với đặc quyền root.
- Điểm cắt đứt của lệnh `USER appuser`:
  Lệnh `USER` cắt đứt chuỗi tấn công ngay tại bước 3. Khi chuyển sang người dùng không có đặc quyền `appuser` (UID 10001), tiến trình bị chiếm quyền chỉ có quyền đọc/ghi hạn chế trong thư mục app. Tiến trình này không có các Linux capabilities đặc quyền (như `CAP_SYS_ADMIN`), không thể cài cắm rootkit, không sửa được file hệ thống của container và loại bỏ khả năng khai thác hầu hết các kỹ thuật container escape để tấn công máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa trong 2 giây liên tiếp: **20 request**.
- Cách đạt được con số đó:
  Với thuật toán fixed window reset lúc giây `00` mỗi phút:
  1. Người dùng gửi liên tục 10 request vào giây `10:00:59` (giây cuối cùng của phút thứ 10:00). Vì hạn mức trong phút 10:00 là 10 request, cả 10 request này đều được hệ thống chấp nhận hợp lệ.
  2. Ngay 1 giây sau đó, đồng hồ chuyển sang `10:01:00`, bộ đếm của hệ thống tự động reset về 0 cho khung giờ mới. Người dùng gửi tiếp 10 request nữa trong giây `10:01:00`. Hệ thống kiểm tra và thấy trong phút 10:01 mới chỉ có 10 request nên tiếp tục cho qua.
  -> Hậu quả là chỉ trong khoảng thời gian 2 giây ngắn ngủi (từ 10:00:59 đến 10:01:00), hệ thống phải hứng chịu một đợt burst lên đến 20 request (gấp đôi hạn mức quy định), dễ gây nghẽn dịch vụ. Thuật toán sliding window 60 giây khắc phục hoàn toàn điều này bằng cách tính tổng số request trong khoảng trượt thực tế `[now - 60, now]`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác biệt cốt lõi:
  - **Rate Limit**: Giới hạn **tần suất và số lượng** request trong một khung thời gian ngắn (ví dụ: tối đa 10 request/phút) nhằm ngăn chặn tấn công DoS, chống nghẽn server và duy trì tính sẵn sàng của hệ thống (trả về HTTP 429).
  - **Cost Guard**: Giới hạn **tổng chi phí tài chính (USD)** trong một chu kỳ dài hạn (ví dụ: 10 USD/tháng) dựa trên số token thực tế tiêu thụ, nhằm bảo vệ ví tiền của nhà phát triển khỏi hóa đơn dịch vụ LLM bất ngờ (trả về HTTP 402).
- Tình huống Rate limit cho qua nhưng Cost guard chặn:
  Một người dùng chỉ gửi 1 request trong cả buổi sáng, nhưng số tiền đã ghi nhận trong tháng là $10.01, vượt ngân sách $10.00. Rate limiter vẫn cho qua vì tần suất thấp, còn `guard.check(user_id)` chặn ngay với HTTP 402. Implementation hiện kiểm tra số tiền đã ghi nhận với `estimated_cost=0`; request làm tổng vừa vượt ngưỡng được ghi chi phí trước, và request kế tiếp mới bị chặn.
- Tình huống Cost guard cho qua nhưng Rate limit chặn:
  Một người dùng mới trong tháng chưa tiêu đồng nào ($0.0 / $10.0). Họ gửi các câu hỏi cực ngắn chỉ tốn $0.00001 mỗi lần, nhưng gửi dồn dập 15 request chỉ trong vòng 10 giây. Về mặt tài chính, chi phí không đáng kể và Cost guard cho phép, nhưng Rate limiter sẽ lập tức chặn từ request thứ 11 với mã lỗi HTTP 429 Too Many Requests vì vi phạm tần suất an toàn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự chuỗi sự kiện thảm họa:
1. Redis gặp sự cố mạng hoặc khởi động lại, tạm thời không thể kết nối trong 30 giây.
2. Cả 3 container agent đều nhận thấy Redis không phản hồi. Vì liveness probe (`/health`) bị gộp kiểm tra Redis, cả 3 container đồng loạt báo trạng thái lỗi HTTP 503 Unhealthy.
3. Bộ điều phối cụm (Orchestrator như Docker, Kubernetes hay Cloud Platform) kiểm tra liveness probe thấy fail nên nhận định rằng tiến trình bên trong container đã bị chết/treo, lập tức gửi tín hiệu cưỡng chế dừng và khởi động lại (restart) cả 3 container cùng lúc.
4. Trong khoảng thời gian các container đang bị restart, hệ thống hoàn toàn không còn bất kỳ instance nào còn sống để nhận traffic, toàn bộ người dùng bên ngoài bị ngắt kết nối hoàn toàn và nhận lỗi 502/503.
5. Khi Redis vừa hồi phục sau 30 giây, các container vẫn đang chật vật trong chu kỳ khởi động hoặc rơi vào trạng thái CrashLoopBackOff. Một sự cố gián đoạn tạm thời ở dịch vụ phụ thuộc (Redis) đã bị khuếch đại thành sự cố sập hoàn toàn toàn bộ cụm ứng dụng (cascading failure).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu lịch sử trong Redis (chuẩn Stateless):
  Tôi cấu hình Nginx đứng trước các replica để cổng host 8000 không bị xung đột khi scale. Tất cả container agent cùng truy cập service Redis nên, dù Nginx phân phối request vào replica nào, `history_length` tăng đều theo số message đã lưu: `0, 2, 4, 6, 8`.
- Nếu lưu trong một dict Python nội bộ (Stateful trong RAM của process):
  Mỗi container sở hữu một vùng nhớ bộ nhớ RAM riêng biệt:
  - Khi request 1 vào container A: dict của A lưu 2 tin nhắn, response trả về `history_length = 0`.
  - Khi request 2 rơi vào container B: dict của B hoàn toàn rỗng, container B không biết gì về cuộc trò chuyện trước đó, agent phản hồi như mới bắt đầu và trả về `history_length = 0` (thay vì 2).
  - Khi request 3 rơi vào container C: dict của C cũng rỗng, trả về `history_length = 0`.
  - Khi request 4 tình cờ rơi lại vào container A: dict của A tìm thấy 2 tin nhắn cũ và trả về `history_length = 2`.
  -> Kết quả là `history_length` sẽ nhảy thất thường, lúc tăng lúc giảm về 0, và agent liên tục bị "mất trí nhớ" tùy thuộc vào việc request ngẫu nhiên chạm vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải**:
  `Health check failed: Timed out waiting for response on port 8000 after 60 seconds.`
- **Cách tìm ra nguyên nhân**:
  Truy cập vào tab Runtime Logs trên giao diện quản lý của dịch vụ cloud (Render/Railway). Xem dòng log khởi động của ứng dụng, tôi thấy uvicorn ghi:
  `INFO: Uvicorn running on http://0.0.0.0:8000`.
  Tuy nhiên, trong tab Variables/Environment của nền tảng, hệ thống cloud tự động sinh và cấp phát một biến môi trường `PORT` (ví dụ `PORT=10000`) và cổng này là nơi router của platform chuyển hướng traffic tới để kiểm tra `/health`. Vì Dockerfile ban đầu cố định lệnh chạy cổng 8000 (`--port 8000`), server không lắng nghe trên cổng mà nền tảng mong muốn, dẫn tới healthcheck timeout.
- **Cách sửa lỗi**:
  Sửa lệnh `CMD` trong `Dockerfile` để đọc linh hoạt biến môi trường `PORT` với giá trị dự phòng là 8000 khi chạy dưới local:
  `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`.
  Sau khi cập nhật và deploy lại, uvicorn khởi động đúng trên cổng do platform chỉ định và health check phản hồi 200 OK ngay lập tức.
