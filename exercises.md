# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng gợi ý bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thùy Trang  Mã học viên: 2A202601559

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tôi đã chạy trực tiếp trong image production không truyền biến môi trường: `python -c "from app.config import Settings; Settings()"`. Kết quả là `pydantic_core._pydantic_core.ValidationError`, trường `agent_api_key`, `Field required`. Vì vậy một revision khởi tạo `Settings` mà thiếu secret sẽ dừng với lỗi cấu hình rõ ràng. Nếu để mặc định `"changeme"`, service vẫn public và người lạ chỉ cần gửi `X-API-Key: changeme` là có thể gọi `/ask`, tiêu quota/chi phí. Fail fast biến lỗi đó thành lỗi deploy thay vì lộ API có khóa đoán được.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Tôi chạy agent + Redis bằng Docker và gọi `/ask` thật. Một dòng từ `docker logs` là: `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:43:35.272071+00:00", "user_id": "exercise-rate-observation", "tokens_in": 467, "tokens_out": 48, "cost_usd": 9.885e-05}`.
>
> Tôi có thể (1) lọc/đếm riêng các event `ask_completed` theo `user_id`, rồi dựng dashboard hay cảnh báo khi số request tăng bất thường; và (2) cộng trường `cost_usd` hoặc phân tích `tokens_in`/`tokens_out` để theo dõi chi phí và phát hiện prompt quá lớn. `print("đã trả lời xong")` chỉ là chuỗi tự do, không có trường dữ liệu ổn định để máy lọc, tổng hợp hoặc cảnh báo đáng tin cậy.

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
| Multi-stage | 309 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch khoảng 1.42 GB. Bản đầu dùng image `python:3.11` đầy đủ, copy toàn bộ context và giữ cả các công cụ/layer phục vụ cài dependency (kể cả dependency test). Bản multi-stage dùng `python:3.11-slim`, chỉ chuyển virtualenv đã cài và hai thư mục runtime `app`, `utils` sang image cuối, nên không mang compiler/cache pip/source không cần thiết. Lưu ý số đo có thể thay đổi theo version image và cache tại thời điểm build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi đã đổi tạm một ký tự trong `app/main.py`, build lại rồi hoàn nguyên. Output Docker cho thấy `WORKDIR`, tạo venv, `COPY requirements.txt`, `RUN pip install`, tạo user và `COPY --from=builder /opt/venv` đều `CACHED`; `COPY --chown=app:app app ./app` và lệnh `COPY utils ./utils` chạy lại. Nếu đặt `COPY . .` trước `RUN pip install`, thay đổi một ký tự ở source sẽ làm layer `COPY` đổi và kéo theo `RUN pip install` chạy lại, dù `requirements.txt` không đổi; build sẽ chậm hơn và dễ phải tải/cài lại thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Tôi kiểm tra container đang chạy bằng `id` và nhận `uid=100(app) gid=101(app) groups=101(app)`, tức process không phải root. Nếu có lỗ hổng như thực thi lệnh/deserialization, kẻ tấn công trước hết chạy code trong process Python. Khi process là root, họ có root trong container, có thể lấy credential hoặc lợi dụng Docker socket/host mount, privileged hay container-escape để leo lên host. `USER app` cắt chuỗi ngay sau bước chạy code: shell bị chiếm chỉ mang uid 100, không tự làm thao tác root trong container. Nó giảm blast radius, dù không thay thế việc vá lỗi và tránh cấu hình container nguy hiểm.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong 2 giây. Người dùng gửi 10 request ở cuối phút, chẳng hạn 10:00:59, rồi gửi tiếp 10 request ngay đầu phút kế, 10:01:00 hoặc 10:01:01. Bộ đếm theo phút đồng hồ reset tại giây 00 nên mỗi nhóm đều được coi là 10/phút, dù thực tế có 20 request trong cửa sổ 2 giây. Sliding window luôn đếm 60 giây gần nhất nên chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Tôi thử rate limit thật với một user mới: 11 status lần lượt là `200` mười lần rồi `429`; request thứ 11 trả `{"detail":"rate limit exceeded"}`. Tôi cũng chạy một container tạm có `MONTHLY_BUDGET_USD=0.00001`: request đầu trả `200` với cost `2.22e-05`, request thứ hai trả `402 monthly budget exceeded`. Điều này cho thấy rate limit giới hạn nhịp request, còn cost guard giới hạn tiền theo tháng. Với code hiện tại, `guard.check(user_id)` không truyền `estimated_cost`, nên request đầu vẫn có thể vượt ngân sách rồi lần sau mới bị chặn; muốn chặn chặt trước LLM cần truyền chi phí ước tính vào `check`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Tôi dừng Redis của đúng stack trong vài giây rồi gọi agent đang chạy: `/health` trả `200 {"status":"ok","service":"day12-agent","version":"1.0.0"}`, còn `/ready` trả `503 {"status":"not ready","redis":false}`. Sau đó tôi start lại Redis và kiểm tra trạng thái `healthy`. Nếu gộp probe và dùng kết quả Redis làm health, cả 3 container sẽ trả 503; orchestrator hiểu nhầm process chết, loại/restart từng container và có thể tạo restart loop trong 30 giây Redis lỗi. Tách hai endpoint giữ process sống, còn load balancer chỉ ngừng gửi traffic nhờ readiness.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Vì cổng 8000 của máy đang do ứng dụng khác giữ, tôi chạy 3 container agent tạm trên cùng Docker network nhưng không publish cổng, rồi gọi luân phiên A → B → C với cùng `X-User-Id`. Các response thật có `history_length` lần lượt `0`, `2`, `4` (cost lần lượt `2.58e-05`, `3.93e-05`, `4.38e-05`), chứng tỏ chúng dùng chung Redis. Nếu thay Redis bằng dict Python, mỗi instance có bộ nhớ riêng nên lần đầu vào A, B, C đều sẽ là 0; các lượt sau sẽ tăng rời rạc theo instance và mất khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp ở bước kiểm tra sau deploy là `{"detail":"There was an error parsing the body"}`, rồi vòng test rate limit trả 401 thay vì 429. Tôi so sánh với request POST JSON UTF-8 bằng Python: API local chạy Docker trả 200 và log `ask_completed`; nguyên nhân là lệnh client trên Windows tạo body/header không đúng, đồng thời vòng lặp chưa có API key hợp lệ, không phải Redis hay cấu hình Render. Tôi đổi sang `curl.exe` hoặc client gửi `json=...` UTF-8 và export key trước khi gọi. Tôi cũng gọi lại URL Render: `/health` trả 200 `{"status":"ok",...}` và `/ready` trả 200 `{"status":"ready","redis":true}`, xác nhận deploy hiện tại đã kết nối Redis.
