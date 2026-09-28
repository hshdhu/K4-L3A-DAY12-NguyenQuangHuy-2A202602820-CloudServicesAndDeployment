# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Quang Huy — Mã học viên: 2A202602820
>

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu tạo service mới trên Render mà quên đặt `AGENT_API_KEY`, việc khởi tạo
`Settings` sẽ báo `ValidationError`, giúp phát hiện cấu hình thiếu trước khi
xử lý request có xác thực. Nếu dùng mặc định `changeme`, service vẫn nhận
khóa dễ đoán đó và người khác có thể gọi API trái phép. Test thiếu API key
của CP1 đã pass. Tuy nhiên, với đường chạy `uvicorn app.main:app` hiện tại,
Settings được lấy khi dependency cần đến; muốn bảo đảm fail fast ngay lúc
startup ở mọi cách chạy thì phải khởi tạo Settings trong lifespan.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log thực tế lấy từ container local sau khi gọi `/ask`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T14:45:18.633391+00:00", "user_id": "reflection-bef06ec2", "tokens_in": 3, "tokens_out": 42, "cost_usd": 2.565e-05}
```

Thứ nhất, có thể lọc theo `user_id` và `timestamp` để tìm các lượt hỏi của
một người dùng trong khoảng thời gian cụ thể. Thứ hai, có thể cộng `cost_usd`
và số token để thống kê mức sử dụng, phát hiện chi phí tăng bất thường.
Chuỗi `đã trả lời xong` không có các trường dữ liệu này để lọc hay tính tổng.

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
| 1 stage (bản đầu) | Chưa đo — cần build bản gốc để bổ sung |
| Multi-stage | 271 MB (`day12-agent:cp2-test`, số đo Docker hiển thị) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Image multi-stage đã build thành công và đạt giới hạn dưới 500 MB.
Chưa có số đo bản một stage nên chưa kết luận được giảm bao nhiêu MB.
Theo cấu trúc Dockerfile, bản mới dùng nền `python:3.11-slim` thay bản Python
đầy đủ, không giữ cache pip và chỉ copy `app`, `utils` cùng dependency vào
runtime. Những thay đổi này giảm thành phần không cần khi chạy. Không thể
quy toàn bộ chênh lệch cho multi-stage, và Dockerfile hiện tại cũng không
cài compiler để có thể nói rằng đã loại compiler khỏi runtime.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Phân tích theo Dockerfile hiện tại: nếu chỉ thay `app/main.py`, stage builder
với `COPY requirements.txt` và `RUN pip install` có thể dùng lại cache.
Các bước runtime trước `COPY app ./app` cũng không đổi. Bước copy app và
các bước phụ thuộc phía sau cần được Docker đánh giá/build lại.
Nếu đưa `COPY . .` lên trước `RUN pip install`, thay source làm mất cache
của bước copy và khiến bước cài thư viện chạy lại, dù requirements không đổi.
Đây là phân tích từ cấu trúc layer; chưa thực hiện thí nghiệm sửa một ký tự
và lưu output cache riêng cho câu này.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Một lỗ hổng thực thi mã có thể cho kẻ tấn công chạy lệnh với quyền của process
Python. Nếu process là root trong container, họ có nhiều quyền hơn để sửa
file, cài công cụ hoặc khai thác cấu hình nguy hiểm. Khi có thêm điều kiện
như mount Docker socket, mount thư mục host nhạy cảm hoặc lỗ hổng thoát
container, cuộc tấn công có thể ảnh hưởng host.
`USER appuser` làm process chạy bằng UID 10001, đã được kiểm tra thực tế,
nên giảm quyền ngay ở bước thực thi mã trong container. Root trong container
không tự động là toàn quyền trên host; non-root cũng không ngăn mọi lỗ hổng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request trong khoảng hai giây bắc qua ranh giới phút: gửi 10
request ngay trước thời điểm reset và 10 request ngay sau đó. Mỗi phút
đồng hồ vẫn chỉ có 10 request. Sliding window luôn xét 60 giây gần nhất,
nên các request ngay trước ranh giới vẫn được đếm và đợt tiếp theo bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tốc độ gửi request, còn cost guard giới hạn tổng tiền
theo user và tháng. Ví dụ một user chỉ gửi 1 request/phút nhưng đã tiêu
10,1 USD với ngân sách 10 USD: rate limit cho qua, cost guard chặn bằng 402.
Ngược lại, user mới tiêu 0,01 USD nhưng gửi request thứ 11 trong 60 giây với
hạn mức 10: ngân sách vẫn còn, nhưng rate limit chặn bằng 429.
Trong code lab, check và ghi chi phí là hai bước riêng; kiểm tra trước không
phải bảo đảm tuyệt đối ngân sách khi có request đồng thời hoặc chi phí mới
chưa được ước lượng.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Redis mất kết nối → cả ba instance kiểm tra endpoint chung và nhận lỗi →
nếu endpoint đó được cấu hình làm liveness và lỗi vượt ngưỡng probe,
orchestrator có thể restart cả ba → trong lúc Redis vẫn lỗi, các instance
mới tiếp tục không qua probe → khi Redis phục hồi, hệ thống còn phải chờ
app khởi động và ổn định. Vì vậy một lỗi dependency có thể gây gián đoạn lớn.
Tách hai endpoint giúp `/health` vẫn báo process sống, còn `/ready` trả 503
để bộ điều phối tạm ngừng chuyển traffic vào instance chưa sẵn sàng.
Hành vi restart/rút traffic phụ thuộc cấu hình nền tảng; Docker Compose
healthcheck đơn thuần không tự restart container chỉ vì unhealthy.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Đã gọi hai request liên tiếp cùng user trên một container local dùng Redis
thật; `history_length` lần lượt là 0 và 2. Giá trị này là số message trước
lượt hỏi hiện tại; mỗi lượt thêm một message user và một message assistant.
Nếu nhiều instance dùng chung Redis, request tuần tự sẽ đọc cùng lịch sử,
tăng theo 0, 2, 4... cho đến giới hạn 20 message.
Nếu lưu bằng dict riêng, request sang instance khác có thể quay về 0 hoặc
một số nhỏ hơn vì mỗi instance có lịch sử khác nhau; restart còn làm mất RAM.
Chưa chạy thí nghiệm ba container thật: Compose hiện publish cùng cổng host,
nên cần điều chỉnh mapping và bố trí load balancer trước khi scale.
Test CP4 mô phỏng hai store dùng chung Redis đã pass, nhưng không thay thế
minh chứng load balancing thực tế.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Chưa ghi nhận lỗi build/runtime trên Render trong lần deploy này, nên không
đưa ra một thông báo lỗi cloud giả. Vướng mắc thực tế ở khâu kiểm tra là
`DEPLOY_API_KEY` còn trống, khiến test gọi `/ask` có xác thực bị skip và
kết quả CP5 lúc đầu là 8 passed, 5 skipped.
Đọc điều kiện skip trong `tests/test_cp5.py` cho thấy biến này phải chứa
API key của service trên cloud, không phải token tài khoản Render.
Sau khi đặt khóa mới trên Render, deploy lại và điền cùng khóa vào
`DEPLOY_API_KEY` cục bộ, CP5 đạt 9 passed, 4 skipped.
Đây là vấn đề cấu hình kiểm thử, chưa đáp ứng hoàn toàn yêu cầu kể một lỗi
deploy. Nếu cần đúng loại tình huống đó, phải bổ sung trải nghiệm thực tế
hoặc trao đổi với Lab Coach; không bịa sự cố để điền bài.
