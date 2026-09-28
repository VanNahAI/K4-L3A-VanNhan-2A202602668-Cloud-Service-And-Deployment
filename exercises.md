# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Chu Văn Nhân  Mã học viên: 2A202602668

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: Khi deploy service lên cloud (Railway/Render) nhưng quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard.
> - Nếu để mặc định `"changeme"`: Service vẫn khởi động và báo healthy bình thường. Bất kỳ ai trên Internet hoặc bot quét web đều có thể gọi API bằng key mặc định `"changeme"`, spam các câu hỏi dài và làm cạn kiệt ngân sách/quota LLM trước khi bạn kịp nhận ra.
> - Khi không có mặc định (Fail Fast): Service crash ngay từ startup với lỗi `ValidationError`. Container dừng lại, orchestrator báo lỗi deploy ngay lập tức giúp ta phát hiện và bổ sung secret trước khi mở traffic ra công chúng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:15:30.123456+00:00", "user_id": "sv-123", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.000288}`
>
> Hai việc làm được với log JSON có cấu trúc mà `print()` văn bản không làm được:
> 1. **Truy vấn, lọc và cảnh báo tự động:** Các hệ thống phân tích log tập trung (Datadog, Grafana Loki, CloudWatch) có thể parse trường JSON tự động để thiết lập dashboard theo dõi số token tiêu thụ, chi phí `cost_usd` theo từng `user_id`, hoặc bắn alert khi chi phí vượt ngưỡng.
> 2. **Phân tích số liệu và thống kê (Aggregation):** Có thể chạy truy vấn tính tổng chi phí theo thời gian, tính latency trung bình, hoặc nhóm theo `user_id` để biết user nào gọi nhiều nhất mà không cần viết regex để bóc tách chuỗi phức tạp.

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
| 1 stage (bản đầu) | ~1.1 GB |
| Multi-stage | ~271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~800MB) bao gồm:
> - Base image ban đầu (`python:3.11` full) chứa toàn bộ bộ công cụ build của hệ điều hành Linux (gcc, g++, make, build-essential, header files, man pages, git, v.v.).
> - File cache sinh ra trong quá trình `pip install` (wheels tạm, source tarballs tải về).
> Trong bản Multi-stage, stage builder sử dụng `python:3.11-slim` và cài đặt với `--no-cache-dir --prefix=/install`. Stage runtime cuối cùng chỉ copy đúng các package đã được build hoàn chỉnh sang một image `python:3.11-slim` sạch sẽ, loại bỏ hoàn toàn compiler và file tạm.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - **Với Dockerfile hiện tại:**
>   - Các layer được dùng lại từ cache: `FROM`, `WORKDIR /app`, `COPY requirements.txt .`, và `RUN pip install ...` (bước tải & cài đặt thư viện được cache hoàn toàn).
>   - Layer phải chạy lại: Bắt đầu từ lệnh `COPY . .` ở stage runtime cho đến cuối (chỉ mất ~1-2 giây).
> - **Nếu đặt `COPY . .` trước `RUN pip install`:**
>   Mỗi khi sửa dù chỉ 1 ký tự trong source code, checksum của build context thay đổi làm mất cache ở layer `COPY . .`. Do đó Docker buộc phải chạy lại toàn bộ lệnh `RUN pip install` phía sau, tải và cài đặt lại toàn bộ dependencies từ đầu mỗi lần build, làm tăng thời gian build từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> - **Chuỗi sự kiện khi chạy với Root:**
>   1. App Python có lỗ hổng (như Remote Code Execution, insecure deserialization, hoặc package chứa backdoor).
>   2. Kẻ tấn công khai thác RCE để thực thi shell payload bên trong container.
>   3. Vì container process chạy quyền root (UID 0), kẻ tấn công là root bên trong container namespace.
>   4. Nếu có lỗ hổng container escape (kernel exploit, misconfigured volume mount `/var/run/docker.sock` hoặc privileges), quyền root trong container chuyển thẳng thành quyền `root` (UID 0) trên máy host -> Kẻ tấn công kiểm soát toàn bộ máy chủ vật lý/host OS.
> - **Lệnh `USER appuser` cắt đứt chuỗi:**
>   Quy định tiến trình Python chạy dưới quyền user thường (UID 10001) không có quyền sudo. Khi bị RCE, kẻ tấn công chỉ có quyền hạn chế của `appuser`, không thể ghi đè file hệ thống, không thể load kernel module, và triệt tiêu khả năng leo thang đặc quyền container escape lên root host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window (ZSET) thay vì fixed window (INCR theo
phút). Mô tả một kịch bản tấn công qua mặt được fixed window nhưng bị sliding
window chặn lại.

> - **Kịch bản tấn công qua mặt Fixed Window:**
>   Giả sử giới hạn là 10 request/phút (reset theo phút tròn).
>   - Kẻ tấn công gửi 10 request ở giây thứ `10:00:59` (thuộc phút 10:00).
>   - Kẻ tấn công gửi tiếp 10 request ở giây thứ `10:01:01` (thuộc phút 10:01).
>   - Với Fixed Window (INCR theo phút): Cả 2 đợt đều hợp lệ vì mỗi phút chỉ có 10 request. Kết quả: Service phải chịu tải 20 request chỉ trong vòng 2 giây (gấp đôi tải cho phép).
> - **Sliding Window ngăn chặn:**
>   Sliding Window đếm số request trong 60 giây gần nhất tính từ thời điểm hiện tại `[now - 60, now]`. Tại thời điểm `10:01:01`, cửa sổ tính từ `10:00:01` đến `10:01:01` vẫn chứa 10 request vừa gửi ở giây `10:00:59`. Request thứ 11 gửi tới sẽ bị trả mã lỗi `429 Too Many Requests` ngay lập tức.

---

### Câu 7 — Ngân sách (CP3)

Một user có rate limit 10 req/phút nhưng vẫn làm hóa đơn LLM tăng vọt $500
trong một đêm. Điều gì đã xảy ra, và `CostGuard` ngăn chặn chuyện này như thế nào?

> - **Điều đã xảy ra:**
>   Rate limiter chỉ kiểm soát **số lượng request**, không kiểm soát **kích thước payload / số token**.
>   User đó gửi 10 request/phút liên tục suốt đêm (khoảng 8 tiếng = 4,800 request). Mỗi request gửi vào context prompt cực dài (ví dụ 50,000 - 100,000 tokens input/output). Với 4,800 request dung lượng lớn, chi phí API LLM tích lũy lên hàng trăm USD dù không hề vi phạm rate limit.
> - **Cách `CostGuard` ngăn chặn:**
>   `CostGuard` theo dõi tổng chi phí USD mà user đã tiêu trong tháng (`cost:<user>:<YYYY-MM>`). Trước mỗi lượt gọi LLM, `CostGuard.check()` kiểm tra xem tổng tiền đã tiêu cộng với chi phí ước tính có vượt qua `MONTHLY_BUDGET_USD` (ví dụ $10.0) hay không. Khi user chạm mốc $10.0, mọi request tiếp theo bị chặn ngay lập tức với mã `402 Payment Required` trước khi request kịp chuyển tới model LLM.

---

### Câu 8 — Mất trí nhớ ngẫu nhiên (CP4)

Hệ thống chạy 3 instance sau Nginx. User A chat: câu 1 được trả lời đúng ngữ
cảnh, câu 2 agent trả lời như chưa từng gặp User A. Giải thích nguyên nhân
gốc rễ và cách `ConversationStore` giải quyết.

> - **Nguyên nhân gốc rễ (Stateful in RAM):**
>   Lịch sử hội thoại được lưu trong một biến/dict nội bộ trong RAM của tiến trình Python. Khi hệ thống có 3 instance (A, B, C) đứng sau Load Balancer:
>   - Câu hỏi 1 của User A được Load Balancer định tuyến vào Instance A -> Instance A lưu lịch sử vào RAM của nó.
>   - Câu hỏi 2 của User A được Load Balancer định tuyến sang Instance B (theo thuật toán Round Robin) -> Instance B không có dữ liệu trong RAM của mình nên coi User A là người lạ hoàn toàn, dẫn đến tình trạng "mất trí nhớ".
> - **Cách `ConversationStore` giải quyết:**
>   Chuyển toàn bộ state ra ngoài tiến trình (Stateless architecture) và lưu tập trung vào Redis chung (`history:<user_id>`). Khi đó, dù Load Balancer chuyển request tới bất kỳ instance nào (A, B hay C), instance đó đều đọc và ghi cùng một danh sách hội thoại trong Redis, đảm bảo ngữ cảnh liên tục và nhất quán.

---

### Câu 9 — Vì sao tách `/health` và `/ready` (CP4)

Redis sập trong 30 giây rồi tự phục hồi. Mô tả điều gì xảy ra nếu orchestrator
chỉ dùng một endpoint chung kiểm tra cả process lẫn Redis. So sánh với khi
tách riêng `/health` và `/ready`.

> - **Nếu dùng một endpoint chung (gộp liveness và dependency check):**
>   Khi Redis mất kết nối 30s, endpoint chung trả về lỗi 503. Orchestrator hiểu nhầm là tiến trình app bị treo/hỏng và liên tục **restart (kill & start lại)** toàn bộ các container agent cùng lúc. Khi Redis vừa khởi động lại, các container app đang trong quá trình bị restart/crash-loop nên toàn bộ hệ thống sập hoàn toàn (cascading failure).
> - **Khi tách riêng `/health` (liveness) và `/ready` (readiness):**
>   - `/health` chỉ kiểm tra process Python còn sống hay không (không phụ thuộc Redis) -> Vẫn trả về 200 OK -> Orchestrator **không restart** container.
>   - `/ready` kiểm tra kết nối tới Redis -> Trả về 503 Not Ready -> Load Balancer tạm thời **ngừng điều hướng traffic** tới container đó để tránh trả lỗi 500 cho user.
>   - Khi Redis hồi phục sau 30s, `/ready` tự động trả lại 200 OK và Load Balancer ngay lập tức mở lại traffic mà không cần restart tiến trình nào.

---

### Câu 10 — Graceful shutdown (CP4)

Bạn bấm deploy phiên bản mới. Mô tả hành trình của một request đến đúng lúc
container cũ nhận `SIGTERM`.

> 1. Khi kích hoạt deploy, Orchestrator gửi tín hiệu **`SIGTERM`** tới container cũ và khởi tạo container mới.
> 2. `Lifecycle` trong container cũ nhận `SIGTERM`, chuyển cờ `shutting_down = True` và chuyển tiếp tín hiệu cho handler của Uvicorn.
> 3. Ngay lập tức, endpoint `/health` và `/ready` trên container cũ trả về mã **503 `shutting_down`**.
> 4. Load Balancer phát hiện mã 503 từ probe và ngay lập tức **rút container cũ ra khỏi danh sách nhận request mới**, chuyển hướng các request mới sang container phiên bản mới.
> 5. **Đối với request đang xử lý dở:** Uvicorn không bị kill đột ngột mà được cấp một khoảng thời gian chờ (grace period 10-30s) để xử lý nốt request hiện tại, ghi nhận xong dữ liệu vào Redis và trả về kết quả 200 trọn vẹn cho người dùng.
> 6. Sau khi hoàn tất request cuối cùng, tiến trình container cũ đóng kết nối và thoát sạch sẽ (exit 0) mà người dùng không gặp bất kỳ lỗi gián đoạn 502/504 nào.
