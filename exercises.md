# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần đánh dấu trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Bùi Văn Khởi  —  Mã học viên: 2A202601723

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu deploy lên cloud mà quên đặt `AGENT_API_KEY`, Settings sẽ báo thiếu biến và service không khởi động. Nhờ fail fast, mình phát hiện lỗi ngay trong lúc deploy, thay vì để app chạy với khóa `changeme` và vô tình cho người khác gọi API tốn phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ: `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T04:00:00+00:00","user_id":"sv-test","cost_usd":0.0001}`. Từ log này có thể lọc tổng chi phí theo `user_id` và đếm tỷ lệ lỗi theo thời gian. `print()` thường chỉ là một chuỗi không có trường dữ liệu để máy tự truy vấn.

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

> Mình chưa đo được hai image bằng `docker images` vì Docker daemon trên máy bị từ chối kết nối, nên không ghi số MB giả. Bản multi-stage chắc chắn nhỏ hơn vì image runtime chỉ giữ Python slim và dependency đã cài; compiler, cache pip và các file build của stage builder không được đưa sang runtime.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, layer `COPY requirements.txt` và `RUN pip install` được dùng lại từ cache; các layer copy source trở đi phải build lại. Nếu đặt `COPY . .` trước `pip install`, mọi thay đổi nhỏ trong source làm cache bị invalid và Docker phải cài lại toàn bộ dependency, khiến build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu app chạy root bị khai thác, tiến trình tấn công có thể đọc/sửa file hệ thống, cài phần mềm hoặc dùng quyền root trong container để tìm đường ảnh hưởng host. Lệnh `USER appuser` làm tiến trình chỉ chạy với UID thường, nên ngay cả khi code bị khai thác thì quyền của kẻ tấn công bị giới hạn ở user đó.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với cách reset theo phút đồng hồ, người dùng có thể gửi 10 request ở giây 59 của phút thứ nhất và thêm 10 request ở giây 00 của phút kế tiếp. Như vậy có tối đa 20 request chỉ trong khoảng 2 giây, dù giới hạn là 10 request/phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lần gọi trong một cửa sổ thời gian, còn cost guard giới hạn tổng tiền theo user trong tháng. Rate limit có thể cho qua 10 request nhỏ nhưng cost guard phải chặn nếu các request trước đã gần hết ngân sách. Ngược lại, khi user còn ngân sách nhưng gửi dồn quá nhanh, rate limit trả 429 còn cost guard vẫn có thể cho phép.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng kiểm tra Redis, Redis mất 30 giây sẽ làm cả ba container trả 503 liveness. Orchestrator hiểu đó là process chết và restart cả ba container, dù code ứng dụng vẫn chạy. Khi tách `/health` và `/ready`, `/health` vẫn 200 nên container không bị restart; chỉ `/ready` trả 503 để load balancer tạm ngừng gửi traffic trong lúc Redis lỗi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, các request đi qua những container khác nhau vẫn đọc cùng một list nên `history_length` tăng đều, ví dụ 0 rồi 2 rồi 4. Nếu dùng dict trong RAM, mỗi container có lịch sử riêng; khi request đổi container, con số có thể quay về 0 hoặc tăng không đều, khiến agent mất ngữ cảnh ngẫu nhiên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi đầu tiên là Uvicorn nhận literal `$PORT` và báo `Invalid value for '--port'`. Mình xem Deploy Logs, thấy Railway đang dùng `startCommand` cũ trong `railway.toml`. Mình bỏ `startCommand` để dùng CMD trong Dockerfile với `${PORT:-8000}`, push lại, rồi Railway deploy thành công. Sau đó `/health` trả 200 và `/ready` trả `{"status":"ready","redis":true}`.
