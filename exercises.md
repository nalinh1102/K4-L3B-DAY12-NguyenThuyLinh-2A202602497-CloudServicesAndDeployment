# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thùy Linh  Mã học viên: 2A202602497

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là khi deploy lên Railway nhưng quên tạo biến
> `AGENT_API_KEY`. Nếu ứng dụng dùng mặc định `"changeme"`, service vẫn báo
> chạy thành công và người ngoài có thể đoán khóa để gọi `/ask`. Với cấu hình
> hiện tại, Pydantic báo thiếu trường bắt buộc ngay lúc khởi động, nên tôi phát
> hiện sai cấu hình trước khi service nhận request và phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log tôi nhận được sau khi gọi `/ask` là:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:48:13.964912+00:00", "user_id": "exercise-log", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}`.
> Từ dòng JSON này, tôi có thể lọc các request theo `event` hoặc `user_id` để
> điều tra lỗi, đồng thời tổng hợp `cost_usd`, số token và timestamp để lập
> biểu đồ hoặc cảnh báo. Một dòng `print("đã trả lời xong")` không có các
> trường có cấu trúc để máy tự lọc và thống kê như vậy.

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
| 1 stage (bản đầu) | 1.696 MB |
| Multi-stage | 305 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại hai image và lấy kích thước bằng `docker image inspect`:
> `agent:single` là 1.695.971.217 byte, còn `agent:multi` là 305.100.758 byte.
> Phần chênh lệch chủ yếu đến từ image `python:3.11` đầy đủ của bản một stage
> và các thành phần không cần thiết khi chạy thật. Bản multi-stage dùng
> `python:3.11-slim`; runtime chỉ nhận virtual environment đã cài dependency
> và mã ứng dụng, không mang theo toàn bộ nội dung của builder.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer tạo base image, virtual environment,
> `COPY requirements.txt` và `pip install` vẫn được lấy từ cache vì file
> dependency không đổi. Layer `COPY app ./app` bị tạo lại, còn các chỉ thị sau
> đó được Docker xử lý lại nhưng rất nhẹ. Nếu đặt `COPY . .` trước `RUN pip
> install`, chỉ một thay đổi trong source cũng làm layer `COPY` đổi và khiến
> Docker phải tải, cài lại toàn bộ dependency, nên build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công trước hết có
> quyền của tiến trình đang chạy trong container. Khi tiến trình là root, kết
> hợp với cấu hình nguy hiểm như mount Docker socket, cấp capability mạnh hoặc
> một lỗ hổng container escape, họ có thể sửa file hệ thống và tiến tới quyền
> cao trên host. `USER app` cắt chuỗi này ở bước chiếm tiến trình: mã độc chỉ
> nhận UID không đặc quyền, nên phạm vi file và thao tác được phép bị thu hẹp.
> Cách này giảm rủi ro nhưng không thay thế việc vá lỗ hổng và giới hạn mount.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong hai giây liên tiếp. Họ gửi 10
> request ở giây cuối của phút hiện tại, ví dụ 12:00:59, rồi gửi thêm 10 request
> ngay sau khi bộ đếm reset ở 12:01:00. Sliding window 60 giây không có ranh
> giới reset này nên sẽ nhìn thấy cả hai nhóm trong cùng cửa sổ và chặn nhóm
> vượt hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất gọi trong một khoảng thời gian ngắn, còn cost
> guard giới hạn chi phí cộng dồn theo người dùng. Một người chỉ gửi một
> request nhưng request đó làm tổng chi phí vượt ngân sách thì rate limit vẫn
> cho qua, còn cost guard phải chặn bằng 402. Ngược lại, người dùng gửi request
> thứ 11 rất rẻ trong một phút khi vẫn còn nhiều ngân sách thì cost guard cho
> phép nhưng rate limit chặn bằng 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng kiểm tra Redis, khi Redis mất kết nối thì cả ba container
> đồng loạt báo không khỏe. Platform sẽ restart chúng, nhưng Redis vẫn chưa
> phục hồi nên các container mới lại fail health check và tiếp tục bị restart,
> làm gián đoạn lớn hơn và xóa thời gian warm-up. Khi tách hai endpoint,
> `/ready` trả 503 để platform tạm ngừng chuyển traffic vào instance, còn
> `/health` vẫn trả 200 vì tiến trình Python vẫn sống. Sau khi Redis hoạt động
> lại, readiness trở về 200 và các instance nhận traffic mà không cần restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, ba instance đọc cùng một lịch sử nên `history_length`
> tăng nhất quán sau mỗi lần gọi dù request rơi vào container nào. Nếu dùng
> một `dict` Python, mỗi container chỉ thấy các request từng tới chính nó. Khi
> load balancer chuyển request giữa ba container, tôi sẽ thấy
> `history_length` lặp lại hoặc tăng giảm không liên tục, ví dụ 0, 0, 2, 0, 2,
> thay vì một chuỗi tăng dần. Khi một container restart, phần lịch sử trong
> dict của nó cũng mất hoàn toàn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi kiểm tra service cloud, `pytest` ban đầu báo `ConnectError: [WinError
> 10013]` cho cả `/health`, `/ready` và `/ask`. Tôi mở đúng public URL trong
> trình duyệt và thấy `/health` vẫn trả `{"status":"ok"}`, nên kiểm tra tiếp
> quyền mạng của tiến trình chạy test thay vì sửa ứng dụng Railway. Sau khi
> chạy test trong môi trường được phép truy cập Internet, CP5 đạt 9 test và 4
> test local fallback được bỏ qua. Nguyên nhân là quyền socket của môi trường
> kiểm tra cục bộ, không phải URL, Redis hay `$PORT` trên Railway.
