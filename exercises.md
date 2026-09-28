# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phạm Hồ Quang Dũng  Mã học viên: 2A202602860

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu tôi quên đặt `AGENT_API_KEY` trên Railway mà chương trình vẫn dùng khóa
> mặc định `"changeme"`, bất kỳ ai đoán được khóa đó đều có thể gọi `/ask` và
> tiêu tài nguyên của service. Khi `agent_api_key` không có mặc định, ứng dụng
> báo lỗi ngay lúc khởi động. Tôi nhìn thấy lỗi trong deployment log và sửa cấu
> hình trước khi service được đưa ra Internet, thay vì chỉ phát hiện sau khi bị
> gọi trái phép.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log JSON tôi thu được:
>
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:54:05.432494+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 24, "cost_usd": 3.6e-05}`
>
> Với log này, tôi có thể lọc tất cả sự kiện `ask_completed` của một `user_id`
> cụ thể và cộng `cost_usd` để theo dõi chi phí. Tôi cũng có thể đếm sự kiện
> theo khoảng thời gian hoặc theo `level` để tạo biểu đồ và cảnh báo. Một câu
> `print("đã trả lời xong")` không có các trường ổn định để máy thực hiện những
> truy vấn đó.

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
| 1 stage (bản đầu) | 1696 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build hai image từ cùng source và đo được bản one-stage khoảng 1696 MB,
> còn bản multi-stage khoảng 271 MB, giảm khoảng 1425 MB. Phần chênh lệch chủ
> yếu đến từ image `python:3.11` đầy đủ chứa nhiều gói hệ điều hành và công cụ
> không cần ở runtime. Bản mới dùng `python:3.11-slim`, cài dependency ở stage
> builder rồi chỉ copy kết quả `/install` sang runtime, nên không mang theo
> môi trường build và các thành phần dư thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `WORKDIR`, copy
> `requirements.txt`, `pip install` và copy dependency từ builder vẫn dùng lại
> cache. Layer `COPY app ./app` thay đổi nên nó và các layer đứng sau phải được
> tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, thay đổi một ký tự trong
> source sẽ làm layer copy đổi, kéo theo cache của bước cài dependency bị mất;
> Docker phải tải và cài lại toàn bộ thư viện dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công trước tiên có
> shell trong container. Nếu process chạy bằng root, shell đó cũng có UID 0
> trong container; kết hợp thêm lỗi kernel/runtime, capability nguy hiểm hoặc
> volume nhạy cảm có thể giúp tác động tới host. `USER appuser` cắt chuỗi tại
> bước thực thi lệnh: mã bị chiếm quyền chỉ chạy với UID 10001 và không có quyền
> sửa file hệ thống hay thực hiện các thao tác đặc quyền trong container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong khoảng hai giây: gửi 10 request
> ngay trước giây 00, ví dụ từ `10:00:59`, rồi gửi tiếp 10 request ngay sau khi
> bộ đếm của phút mới reset ở `10:01:00`. Mỗi phút lịch chỉ ghi nhận 10 request
> nên đều hợp lệ, nhưng thực tế hệ thống nhận một burst 20 request. Sliding
> window nhìn 60 giây gần nhất nên chặn được cách lách ranh giới này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ hoặc số request trong 60 giây, còn cost guard giới
> hạn tổng số tiền một user đã tiêu trong tháng. Một user gửi rất ít request
> nhưng mỗi request có prompt/response lớn có thể vẫn nằm dưới rate limit nhưng
> bị cost guard chặn vì hết ngân sách. Ngược lại, một loạt request rất rẻ trong
> vài giây có thể chưa đáng kể về chi phí nên cost guard cho qua, nhưng rate
> limit phải trả 429 để bảo vệ năng lực phục vụ.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng kiểm tra Redis, khi Redis mất kết nối thì cả ba container
> cùng trả health check lỗi. Orchestrator coi cả ba process đã chết và restart
> chúng gần như đồng thời. Các container mới vẫn không nối được Redis nên tiếp
> tục fail và rơi vào vòng lặp restart, khiến cụm không còn instance ổn định để
> phục vụ cả những chức năng không cần Redis. Khi Redis trở lại, ứng dụng còn
> phải chờ vòng restart và khởi động lại. Tách `/health` và `/ready` giúp process
> vẫn sống, còn load balancer chỉ tạm ngừng gửi traffic cho tới khi Redis phục
> hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Trong kiểm thử với hai `ConversationStore` dùng chung Redis, request đầu có
> `history_length = 0` và request tiếp theo thấy hai message trước đó nên có
> `history_length = 2`. Khi nhiều container cùng dùng Redis, độ dài sẽ tiếp tục
> tăng nhất quán bất kể request rơi vào instance nào. Nếu mỗi container dùng
> một dict Python riêng, request qua các instance khác nhau sẽ thấy các bản lịch
> sử khác nhau; số có thể lặp hoặc nhảy như `0, 0, 2, 0, 2...`, và reset về 0
> khi container bị restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi chạy stack container để chuẩn bị deploy, service `agent` hiện trạng thái
> `Exited (3)` dù image build thành công. Tôi dùng `docker compose ps -a` rồi
> `docker compose logs agent` và thấy lỗi
> `NotImplementedError: TODO (CP4): cài đặt install` tại
> `lifecycle.install()` trong lúc FastAPI startup. Nguyên nhân là lifespan đã
> gọi phần graceful-shutdown nhưng hai signal handler còn để TODO. Tôi cài đặt
> `install()` cho `SIGTERM`/`SIGINT`, cài đặt `request_shutdown()` để bật cờ và
> gọi handler cũ, sau đó build lại. Container chuyển sang `Online`; trên Railway
> `/health` và `/ready` đều trả 200.
