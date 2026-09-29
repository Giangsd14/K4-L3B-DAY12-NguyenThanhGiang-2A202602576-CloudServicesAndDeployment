# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bên dưới bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thanh Giang  Mã học viên: 2A202602576

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Render, nếu tôi quên không set biến `AGENT_API_KEY` trong dashboard, app sẽ crash ngay lúc khởi động với lỗi `ValidationError`. Tôi nhìn vào log thấy rõ nguyên nhân, fix ngay trong vòng 1 phút. Nếu để mặc định `"changeme"`, app sẽ chạy bình thường — nhưng bất kỳ ai biết giá trị đó đều gọi được API của tôi thoải mái, tốn tiền LLM mà tôi không biết cho tới khi nhận hóa đơn cuối tháng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được khi gọi `/ask`: `{"timestamp": "2026-09-29T15:52:13Z", "event": "ask_completed", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`
>
> Hai việc làm được với log JSON mà `print` không làm được:
> 1. **Lọc và tổng hợp tự động**: Có thể dùng `jq` hoặc đẩy vào Datadog để lọc tất cả dòng có `cost_usd > 0.001` trong 1 giờ, tính tổng chi phí theo từng `user_id` — điều không thể làm với chuỗi text thuần.
> 2. **Cảnh báo theo ngưỡng**: Hệ thống log có thể đọc trường `tokens_out` và tự động bắn alert khi một user dùng quá nhiều token. `print` chỉ in ra màn hình, không trigger được cảnh báo.

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
| 1 stage (bản đầu) | ~280 MB (ước tính, chưa build riêng) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch là các công cụ build-time không cần thiết khi chạy: compiler, header file C, build tools mà `pip` dùng để compile thư viện. Stage `builder` cài chúng để build wheel, nhưng stage `runtime` chỉ copy kết quả đã compile sang — bỏ lại toàn bộ "giàn giáo" sau khi nhà đã xong.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại (`COPY requirements.txt` → `RUN pip install` → `COPY app`): khi sửa `main.py`, Docker dùng lại cache đến hết bước `pip install` vì `requirements.txt` không đổi, chỉ re-run từ bước `COPY app` trở đi — build xong trong vài giây.
>
> Nếu đặt `COPY . .` trước `RUN pip install`: mỗi lần sửa bất kỳ dòng code nào, layer `COPY` bị invalidate → buộc chạy lại `pip install` từ đầu, tốn 2-3 phút dù requirements không thay đổi gì.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện nếu chạy bằng root: (1) Kẻ tấn công khai thác lỗ hổng path traversal trong code Python. (2) Process đang chạy bằng root → hắn đọc được `/etc/shadow`, ghi đè file hệ thống trong container. (3) Nếu có lỗ hổng container escape hoặc volume mount, quyền root trong container leo thang thành root trên máy host — toàn bộ máy chủ bị kiểm soát.
>
> Lệnh `USER appuser` cắt đứt ở bước (2): dù khai thác được lỗ hổng code, process chỉ có quyền `appuser` (uid 10001) — không ghi được `/etc`, không đọc file user khác, không leo thang quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với fixed window, user có thể gửi tối đa **20 request trong 2 giây**. Cách đạt: gửi 10 request vào giây 59 của phút cũ (hết quota), rồi ngay lập tức gửi thêm 10 vào giây 00 của phút mới (quota vừa reset). Trong 2 giây đó, server nhận 20 request — gấp đôi hạn mức — mà không bị chặn.
>
> Sliding window không bị vấn đề này: nó luôn nhìn lại đúng 60 giây tính từ thời điểm hiện tại, không bao giờ có "cửa sổ nối nhau" để lợi dụng.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác nhau**: Rate limit kiểm soát **tần suất** (số request/phút), cost guard kiểm soát **ngân sách** (tổng tiền/tháng). Cái trước bảo vệ hạ tầng khỏi overload, cái sau bảo vệ ví tiền.
>
> **Rate limit cho qua, cost guard chặn**: User gửi 1 câu mỗi giờ (rất thưa, không vi phạm rate limit), nhưng mỗi câu sinh ra 1000 token. Sau vài trăm câu trong tháng, tổng chi phí vượt ngân sách — cost guard chặn.
>
> **Cost guard cho qua, rate limit chặn**: User còn nhiều budget nhưng đột nhiên gọi 15 lần trong 1 phút do bug client tự retry. Rate limit phát hiện và trả 429, dù budget vẫn còn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện: (1) Redis mất kết nối. (2) `/health` của cả 3 container fail. (3) Orchestrator thấy liveness probe thất bại → restart tất cả container. (4) Trong lúc restart không instance nào phục vụ được → **downtime toàn cụm**. (5) Container mới khởi động, `/health` vẫn fail vì Redis chưa back → bị kill tiếp, vòng lặp restart vô tận.
>
> Tách `/health` (chỉ check process) và `/ready` (check Redis): khi Redis chết, `/health` vẫn xanh → container không bị restart. `/ready` trả 503 → load balancer ngừng đẩy traffic mới, nhưng không kill container. Sau 30 giây Redis hồi phục, mọi thứ về bình thường. Zero downtime.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lịch sử lưu trong dict Python trong RAM mỗi container: load balancer phân phối request round-robin. Container A nhớ 5 tin nhắn của user, nhưng request tiếp theo vào container B thì `history_length` trả về 0 (B không có gì). Rồi vào A lại thấy 5, vào C thấy 0... Con số nhảy lộn xộn: 0, 0, 0, 1, 0, 0, 2 — không tăng đều.
>
> Với Redis, tất cả 3 container dùng chung một nơi lưu trữ, nên `history_length` tăng đều: 1, 2, 3, 4, 5... bất kể request vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi gặp phải**: Sau khi Render báo "Deploy succeeded | Live", truy cập `/ready` và `/ask` đều trả về `500 Internal Server Error`.
>
> **Cách tìm nguyên nhân**: Dùng `curl` gọi lần lượt từng endpoint. `/health` trả 200 bình thường, nhưng `/ready` và `/ask` đều 500. Nhận ra điểm chung: cả hai đều gọi `get_settings()` để lấy config, còn `/health` thì không. Pydantic raise `ValidationError` khi thiếu biến bắt buộc không có default. Kết luận: thiếu biến `AGENT_API_KEY` trên server.
>
> **Cách sửa**: Vào Render Dashboard → Environment → Add Environment Variable, điền `AGENT_API_KEY` → Save → Manual Deploy. Sau khi deploy lại, `/ready` trả `{"status":"ready","redis":true}` và mọi thứ hoạt động bình thường.
