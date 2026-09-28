# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Văn Hưởng Mã học viên:2A202602743

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là khi tôi deploy service lên cloud nhưng quên khai
> báo `AGENT_API_KEY`. Vì trường này không có giá trị mặc định, Pydantic báo
> lỗi ngay khi service khởi động và bản deploy không được đưa ra phục vụ. Tôi
> có thể nhìn log deploy, bổ sung secret rồi chạy lại trước khi có request thật.
> Nếu code dùng mặc định `"changeme"`, service vẫn lên bình thường với một khóa
> rất dễ đoán; người lạ có thể gọi `/ask` và làm phát sinh chi phí mà tôi không
> nhận ra ngay.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Sau khi gọi `/ask` bằng user `sv01`, tôi thu được dòng log:
>
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:00:54.643193+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}`
>
> Từ JSON này, tôi có thể lọc và đếm các sự kiện `ask_completed` theo
> `user_id` hoặc theo khoảng thời gian. Tôi cũng có thể cộng `cost_usd` và số
> token của từng user để tìm người dùng tốn nhiều chi phí hoặc tạo cảnh báo khi
> chi phí tăng bất thường. Dòng `print("đã trả lời xong")` không chứa các trường
> có cấu trúc nên không thực hiện được hai việc đó một cách đáng tin cậy.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản               | Dung lượng |
| ----------------- | ---------- |
| 1 stage (bản đầu) | 1.7 GB     |
| Multi-stage       | 297 MB     |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build hai image `agent:single` và `agent:multi`, sau đó đọc kết quả bằng
> `docker image ls`. Bản một stage dùng image `python:3.11` đầy đủ và giữ toàn
> bộ filesystem của quá trình build trong image cuối. Bản multi-stage dùng
> `python:3.11-slim`; stage runtime chỉ nhận các package đã cài từ builder cùng
> với `app` và `utils`. Vì vậy runtime không mang theo phần hệ điều hành đầy đủ,
> file build và các file khác trong repository. Trong lần đo này, kích thước
> hiển thị giảm từ 1.7 GB xuống 297 MB, chênh khoảng 1.4 GB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi sửa source rồi build lại bản multi-stage, các layer base image,
> `COPY requirements.txt`, `pip install`, tạo `appuser` và copy dependency từ
> builder đều được lấy từ cache vì `requirements.txt` không thay đổi. Layer
> `COPY app ./app` phải chạy lại do nội dung `app/main.py` đã đổi; các layer
> runtime đứng sau nó và bước export image cũng được tạo lại. Nếu đặt
> `COPY . .` trước `RUN pip install`, chỉ một thay đổi trong source cũng làm
> layer `COPY` đổi, kéo theo layer cài dependency mất cache. Khi đó Docker phải
> tải và cài lại toàn bộ package dù `requirements.txt` không đổi, làm build
> chậm hơn rất nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công trước tiên
> chiếm quyền của process trong container. Nếu process chạy bằng root, họ có
> thể sửa file hệ thống trong container, đọc các secret hoặc volume được mount,
> rồi lợi dụng cấu hình đặc quyền, Docker socket hay một lỗ hổng container/kernel
> để tác động đến host với quyền cao. Lệnh `USER appuser` cắt chuỗi này ngay sau
> bước thực thi lệnh: mã độc chỉ chạy dưới UID 10001, không có quyền root để sửa
> file hệ thống hay dùng các tài nguyên đặc quyền. Cơ chế này không thay thế
> việc vá lỗ hổng, nhưng làm giảm đáng kể phạm vi thiệt hại nếu ứng dụng bị phá.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong 2 giây. Cụ thể, họ gửi 10
> request vào cuối phút, chẳng hạn từ `10:00:59` đến ngay trước `10:01:00`.
> Khi đồng hồ sang `10:01:00`, bộ đếm của phút mới được reset nên họ gửi tiếp
> 10 request ngay đầu phút. Như vậy cả hai phút riêng lẻ đều không vượt mức
> 10 request, nhưng thực tế server phải nhận 20 request gần như cùng lúc. Sliding
> window 60 giây ngăn được kẽ hở này vì tại thời điểm nhận nhóm thứ hai, 10
> request của nhóm đầu vẫn còn nằm trong cửa sổ 60 giây gần nhất.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ, tức số request của một user trong cửa sổ 60 giây;
> khi vượt nó trả `429 Too Many Requests`. Cost guard giới hạn tổng số tiền user
> đã tiêu trong tháng; khi vượt ngân sách nó trả `402 Payment Required`.
>
> Ví dụ rate limit cho qua nhưng cost guard chặn: user chỉ gửi request đầu tiên
> trong phút nên vẫn còn quota tốc độ, nhưng trước đó đã tiêu hết ngân sách
> tháng, vì vậy request bị chặn với 402. Trường hợp ngược lại, user còn nguyên
> ngân sách nhưng gửi request thứ 11 trong vòng 60 giây; cost guard vẫn cho qua
> về mặt chi phí, còn rate limiter chặn request đó với 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp hai endpoint và dùng kết quả kiểm tra Redis làm liveness probe, khi
> Redis mất kết nối thì cả ba container agent gần như đồng thời trả 503. Sau
> số lần probe thất bại theo cấu hình, orchestrator cho rằng cả ba process đã
> chết và restart chúng, dù bản thân process Python vẫn hoạt động. Các request
> đang xử lý có thể bị ngắt, còn container mới khởi động vẫn tiếp tục fail nếu
> Redis chưa phục hồi, tạo thành vòng lặp restart và làm cả cụm mất khả dụng.
> Khi Redis hoạt động lại sau 30 giây, các container còn phải khởi động lại và
> qua health check mới nhận traffic được. Với hai endpoint riêng, `/health`
> vẫn trả 200 nên container không bị restart; `/ready` trả 503 để load balancer
> tạm ngừng gửi request. Redis phục hồi thì `/ready` tự trở lại 200 và cả ba
> container được đưa vào phục vụ mà không cần khởi động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, mỗi lần `/ask` ghi hai message gồm câu hỏi của user và
> câu trả lời của assistant. Vì vậy khi gọi liên tiếp với cùng `X-User-Id`,
> tôi thấy `history_length` tăng theo `0, 2, 4, 6, ...` dù request được chuyển
> tới container nào. Nếu dùng một dict Python, ba container có ba bản lịch sử
> độc lập. Với phân phối round-robin, kết quả có thể thành `0, 0, 0, 2, 2, 2,
> ...`; nếu cách phân phối không đều thì con số còn nhảy lên xuống tùy request
> rơi vào instance nào. Khi một container restart, lịch sử trong dict của nó
> mất hoàn toàn và request tới container đó lại thấy `history_length` bằng 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp ở bước deploy là terminal không nhận lệnh Railway và báo
> `zsh: command not found: railway`. Tôi kiểm tra bằng `command -v railway`
> và thấy máy chưa cài Railway CLI, nên lỗi nằm ở công cụ deploy chứ không phải
> Dockerfile hay ứng dụng. Tôi khắc phục bằng cách chạy CLI trực tiếp qua
> `npx --yes @railway/cli`, đăng nhập bằng browserless login, tạo project cùng
> service Redis rồi chạy deploy từ thư mục repository. Sau khi deployment báo
> `SUCCESS`, tôi tạo public domain và kiểm tra lại URL thật: `/health` và
> `/ready` đều trả 200, còn `/ask` không có API key trả 401 như mong đợi.
