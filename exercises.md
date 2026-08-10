# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng hướng dẫn bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Anh Quân  Mã học viên: 2A202601251

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên môi trường Production/Staging, nếu quên khai báo biến môi trường `AGENT_API_KEY` trên Cloud Dashboard: Nếu có giá trị mặc định `"changeme"`, ứng dụng vẫn khởi động thành công và lặng lẽ chạy. Kẻ tấn công hoặc bot tự động quét Internet có thể dò ra endpoint `/ask` và sử dụng API key mặc định `"changeme"` để gọi dịch vụ LLM miễn phí, đốt sạch ngân sách API mà bạn không hay biết. Ngược lại, khi không có giá trị mặc định, Pydantic ném ngay `ValidationError` lúc khởi động container (Fail Fast). Container không khởi động được và lập tức báo đỏ trên màn hình deploy, giúp bạn phát hiện ra thiếu sót ngay từ phút đầu tiên.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T09:30:00+00:00", "user_id": "sv01", "tokens_in": 15, "tokens_out": 45, "cost_usd": 0.0001}`

Hai việc làm được với log JSON:
1. Lọc, nhóm và tính toán chỉ số tự động trên các hệ thống quản lý log (Datadog, Elasticsearch, Grafana Loki) — ví dụ: truy vấn tổng chi phí `cost_usd` hoặc trung bình số token theo từng `user_id` trong khoảng thời gian bất kỳ.
2. Cấu hình tự động gửi cảnh báo (Alerting) khi tỷ lệ sự kiện lỗi (`"level": "error"`) vượt quá ngưỡng cho phép trong 5 phút mà không bị hiện tượng vỡ dòng hay sai lệch cấu trúc dữ liệu.

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
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~835 MB) chứa toàn bộ các công cụ biên dịch (gcc, g++, make, build-essential), các file header C/C++ và bộ nhớ đệm xây dựng trung gian. Những công cụ này rất cần thiết ở bước cài đặt thư viện (`builder`), nhưng hoàn toàn thừa thãi khi ứng dụng thực thi ở môi trường production (`runtime`). Việc tách multi-stage giúp loại bỏ hoàn toàn các công cụ biên dịch này khỏi image thương mại.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa `app/main.py`: Các layer từ `FROM`, `WORKDIR`, `COPY requirements.txt .` và `RUN pip install` đều được dùng lại từ Docker cache. Chỉ có layer `COPY . .` và các lệnh phía sau mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần bạn chỉnh sửa bất kỳ dòng code nào trong ứng dụng, layer `COPY . .` sẽ bị thay đổi và làm hủy bỏ toàn bộ cache của các lệnh phía sau. Docker sẽ buộc phải thực thi lại lệnh `RUN pip install`, kéo dài thời gian build từ 1-2 giây lên vài phút cho mỗi lần thay đổi nhỏ.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python dính lỗ hổng bảo mật (ví dụ: Remote Code Execution - RCE qua `eval()`, `pickle`, hay xử lý fileupload).
2. Kẻ tấn công khai thác lỗ hổng và thực thi mã độc/lệnh shell bên trong container.
3. Nếu container đang chạy bằng quyền `root` (UID 0), mã độc của kẻ tấn công có đầy đủ quyền root bên trong container, từ đó có thể đọc/ghi dữ liệu hệ thống, lạm dụng Docker socket hoặc khai thác lỗ hổng thoát container (container escape) để kiểm soát máy host với quyền cao nhất.
Lệnh `USER appuser` cắt đứt chuỗi ở bước 3: Kẻ tấn công khi xâm nhập thành công chỉ mang quyền hạn của một người dùng thông thường (`appuser`, UID 10001) bị giới hạn tối đa quyền truy cập file và không thể thực thi các lệnh quản trị hệ thống để leo thang quyền lực.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
Cách đạt được:
- Người dùng gửi 10 request vào giây thứ 59 của phút N (lúc 10:00:59).
- Đến giây 00 của phút N+1 (lúc 10:01:00), bộ đếm fixed window tự động reset về 0.
- Người dùng gửi tiếp 10 request nữa vào giây thứ 01 của phút N+1 (lúc 10:01:01).
Kết quả: 20 request được gửi trong khoảng thời gian từ 10:00:59 đến 10:01:01 (2 giây) mà vẫn hợp lệ đối với bộ đếm theo phút cố định, gây ra tải đột biến nguy hiểm cho hệ thống.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt: Rate Limit quản lý **tần suất/số lượng** request trong một cửa sổ thời gian ngắn, trong khi Cost Guard quản lý **tổng chi phí tài chính** phát sinh trong khoảng thời gian dài (tháng).

- Tình huống Rate Limit cho qua nhưng Cost Guard chặn: Người dùng gửi request thứ 2 trong phút (chưa vượt hạn mức 10 req/phút), nhưng câu hỏi chứa prompt cực lớn (100.000 tokens) khiến chi phí vượt qua ngân sách còn lại của tháng ($10.00) -> Cost Guard chặn lại với mã lỗi 402.
- Tình huống Cost Guard cho qua nhưng Rate Limit chặn: Người dùng gửi request thứ 11 liên tiếp trong vòng 10 giây với câu hỏi siêu ngắn ($0.00001, ngân sách tháng vẫn dư nhiều) -> Rate Limit phát hiện vượt quá 10 req/phút nên chặn lại với mã lỗi 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis bị mất kết nối trong 30 giây.
2. Endpoint gộp phát hiện Redis chết và trả về 503 HTTP.
3. Orchestrator (Docker Swarm/K8s/Cloud platform) nhận lỗi 503 từ liveness probe và coi cả 3 container agent đều đã chết, lập tức phát tín hiệu tiêu diệt và khởi động lại (restart) cả 3 container.
4. Trong lúc cả 3 container đang trong quá trình bị tắt và boot lại, hệ thống không còn bất kỳ container nào chạy để xử lý request -> Toàn bộ ứng dụng bị ngắt kết nối hoàn toàn.
5. Khi Redis hồi phục sau 30s, cụm container vẫn còn dở dang trong quá trình restart, làm kéo dài thời gian gián đoạn dịch vụ không cần thiết (Cascading Failure).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong dict Python local RAM: Mỗi request được Load Balancer phân phối ngẫu nhiên tới 1 trong 3 container. `history_length` sẽ thay đổi thất thường không theo thứ tự (ví dụ: request 1 vào A -> len=0, request 2 vào B -> len=0, request 3 vào C -> len=0, request 4 vào A -> len=1). Agent sẽ bị "mất trí nhớ" bất chợt giữa các lượt chat do state bị chia rẽ độc lập giữa RAM các process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: Health Check Timeout / Container startup failed khi deploy lên Cloud Platform (Railway/Render).
Thông báo lỗi: `Application failed to respond to health check on 0.0.0.0:8000`.
Nguyên nhân: Môi trường Cloud tự động cấp phát cổng HTTP thông qua biến môi trường `$PORT` (chẳng hạn PORT=4123), trong khi ứng dụng uvicorn bị gán cứng cổng `8000`.
Cách sửa: Sửa lệnh CMD trong Dockerfile và file `main.py` thành `uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}` để ưu tiên đọc cổng từ biến môi trường của platform.
