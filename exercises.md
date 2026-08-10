# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: dán câu trả lời bên dưới từng câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Kim Duy Hùng  Mã học viên: 01763

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để giá trị mặc định `"changeme"`, khi deploy lên Cloud mà quên cài đặt biến `AGENT_API_KEY`, ứng dụng vẫn khởi động thành công và chạy bình thường. Kẻ gian có thể phát hiện và dùng key mặc định `"changeme"` để gọi API miễn phí, tiêu tốn sạch ngân sách LLM của bạn mà bạn chỉ phát hiện ra khi nhận hóa đơn tài chính cuối tháng. Ngược lại, việc không để mặc định khiến app ném lỗi `ValidationError` dừng ngay lập tức lúc khởi động (Fail-fast), giúp kỹ sư phát hiện và bổ sung ngay trên dashboard Cloud khi vừa deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T10:30:00+00:00", "user_id": "sv-01", "cost_usd": 0.0001, "tokens_in": 12, "tokens_out": 24}`

Hai việc làm được với log JSON:
1. **Lọc và tính toán chi phí tự động**: Các công cụ lưu trữ log (Datadog/Elasticsearch/Loki) tự động parse trường `cost_usd` và `user_id` để truy vấn chính xác "User nào tiêu nhiều tiền nhất hôm nay" bằng truy vấn SQL/KQL.
2. **Cảnh báo lỗi tự động (Automated Alerting)**: Hệ thống giám sát có thể đặt ngưỡng tự động kích hoạt chuông cảnh báo (PagerDuty/Slack notification) nếu số lượng log có `"level": "error"` vượt quá 5% trong 5 phút gần nhất.

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

Phần dung lượng chênh lệch (~835 MB) bao gồm: bộ biên dịch C/C++ (GCC, make), các file header phục vụ việc biên dịch gói thư viện Python, cache cài đặt của `pip`, và các công cụ phát triển thừa của base image đầy đủ `python:3.11`. Trong Multi-stage build, stage `runtime` sử dụng `python:3.11-slim` chỉ copy phần thư viện đã cài đặt sẵn ở đường dẫn `/install` sang, loại bỏ toàn bộ bộ biên dịch và rác build kềnh càng ở lại stage `builder`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại: Các layer base image, `COPY requirements.txt` và `RUN pip install` đều được dùng lại từ cache (chỉ mất ~0.1s). Chỉ có layer `COPY app ./app` và các bước phía sau mới phải chạy lại.
Nếu đặt `COPY . .` lên trước `RUN pip install`: Khi sửa 1 ký tự trong `app/main.py`, Docker phát hiện layer `COPY . .` bị thay đổi nên vô hiệu hóa cache từ dòng đó trở đi. Docker sẽ buộc phải chạy lại lệnh `RUN pip install` tải và cài lại toàn bộ thư viện từ đầu, khiến thời gian build kéo dài hàng phút thay vì 1 giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Kẻ tấn công khai thác lỗ hổng Remote Code Execution (RCE) trong code Python để thực thi lệnh shell bên trong container.
2. Vì container chạy mặc định với quyền `root`, lệnh shell đó có full quyền `root` trong container.
3. Kẻ tấn công dùng kỹ thuật Container Escape (khai thác cgroup, kernel vulnerability hoặc bind mount socket) để thoát khỏi ranh giới container sang máy Host.
4. Vì UID của process bên trong container trùng với UID `root` (0) của máy Host, kẻ tấn công lập tức chiếm quyền quản trị tối cao `root` trên máy Host.

Lệnh `USER appuser` chuyển process sang chạy với UID 1000 (user thường). Nhờ đó, nếu kẻ tấn công thoát được khỏi container thì ở máy Host họ vẫn chỉ là một user không có đặc quyền, cắt đứt hoàn toàn chuỗi tấn công chiếm quyền máy Host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request trong 2 giây liên tiếp**.
Cách đạt được: Người dùng gửi 10 request từ giây `10:00:59` (1 giây cuối của phút thứ 10). Ngay giây tiếp theo `10:01:00`, bộ đếm phút cố định bị reset về 0, người dùng gửi tiếp 10 request ở giây `10:01:01` (1 giây đầu của phút thứ 11). Như vậy cả 2 phút đều ghi nhận đúng 10 request/phút, nhưng thực tế người dùng đã xả dồn dập 20 request chỉ trong khoảng thời gian 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau:
- **Rate Limit**: Giới hạn *số lượng request* gửi tới trong một khoảng thời gian ngắn (ví dụ: 10 request/phút) để bảo vệ server khỏi bị quá tải dồn dập (DDoS).
- **Cost Guard**: Giới hạn *tổng chi phí tài chính (USD)* đã tiêu tốn trong một khoảng thời gian dài (ví dụ: 10.0$/tháng) để tránh cạn kiệt ngân sách do gọi LLM.

Tình huống:
1. **Rate limit cho qua nhưng Cost Guard chặn**: User gửi 1 request duy nhất trong phút (đúng luật rate limit 10/phút), nhưng câu hỏi kéo dài 100,000 token làm chi phí phát sinh là 2.0$ trong khi ngân sách tháng còn lại 0.1$ -> Rate limit duyệt cho qua nhưng Cost Guard chặn (trả 402).
2. **Cost Guard cho qua nhưng Rate limit chặn**: User chưa tiêu đồng nào trong tháng (ví dụ mới tiêu 0$), nhưng gửi dồn dập 15 request ngắn chỉ trong 5 giây -> Cost Guard cho qua vì còn dư ngân sách, nhưng Rate limit sẽ chặn từ request thứ 11 (trả 429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis gặp sự cố mất kết nối trong 30 giây.
2. Endpoint gộp kiểm tra Redis thất bại và trả về HTTP 500/503.
3. Orchestrator (Docker/Kubernetes) hiểu lầm rằng toàn bộ process của container đã bị chết (Liveness failed).
4. Orchestrator lập tức kill và restart lại toàn bộ cụm 3 container cùng một lúc.
5. Khi 3 container mới khởi động lại, Redis vẫn chưa có lại kết nối, nên endpoint lại tiếp tục thất bại và Orchestrator tiếp tục kill/restart liên tục (CrashLoopBackOff).
6. Sự cố nhỏ của Redis bị khuếch đại thành sự cố sập toàn bộ hệ thống ứng dụng. Nếu tách biệt, `/health` vẫn 200 (không restart container) còn `/ready` trả 503 để Load Balancer tạm thời dừng nhận request mới.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử lưu trong dict RAM Python: Lớp Load Balancer sẽ phân phối các request ngẫu nhiên tới 3 instance (Container A, B, C). Con số `history_length` trong response sẽ nhảy thất thường không theo thứ tự tăng dần (ví dụ: 0 -> 0 -> 2 -> 0 -> 4 -> 2...), vì mỗi request rẽ vào một instance có bộ nhớ RAM độc lập. Khi lưu ở Redis, tất cả 3 instance cùng đọc/ghi vào một nguồn chung nên `history_length` luôn tăng đều đặn (0 -> 2 -> 4 -> 6...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: **Health check timeout / Container Crash on Cloud**.
- **Thông báo lỗi**: `Application failed to respond on port 8000. Health check failed.`
- **Cách tìm ra nguyên nhân**: Mở tab *Logs* trên Dashboard của Platform (Railway/Render), quan sát thấy Uvicorn báo lắng nghe thành công ở port `8000`, nhưng Platform lại thực hiện probe kiểm tra ở cổng được gán ngẫu nhiên qua biến `$PORT` (ví dụ `port 5432`).
- **Cách sửa**: Sửa lại lệnh CMD trong `Dockerfile` và code `app/config.py` để ứng dụng đọc linh hoạt cổng từ biến môi trường `PORT` (`uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}`) thay vì cố định cổng 8000.
