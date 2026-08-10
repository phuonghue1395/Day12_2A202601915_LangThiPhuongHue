# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu câu trả lời bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lăng Thị Phương Huế  Mã học viên: 2A202601915

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường trên production khi quên cấu hình biến môi trường `AGENT_API_KEY`. Lúc này, bất kỳ ai gửi request với header `X-API-Key: changeme` cũng có thể gọi thành công các dịch vụ LLM đắt đỏ của mình, dẫn tới việc rò rỉ chi phí nghiêm trọng mà không phát hiện kịp thời. Ngược lại, việc "chết sớm" (Fail Fast) ngay lúc khởi động do thiếu biến môi trường sẽ báo động lỗi lập tức cho nhà phát triển/hệ thống deploy (ví dụ: Railway báo Crash, Kubernetes báo CrashLoopBackOff), giúp ngăn chặn lỗ hổng bảo mật và thất thoát ngân sách ngay trước khi ứng dụng tiếp nhận bất kỳ traffic nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

- Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T15:40:20.671399+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`
- Hai việc có thể làm với dòng log này:
  1. Dễ dàng dùng các công cụ phân tích log tập trung (như ELK stack, Datadog, CloudWatch) để parse cấu trúc tự động, từ đó truy vấn nhanh chóng: "User nào đang tiêu tốn nhiều tiền gọi LLM nhất hôm nay?" hoặc tính tổng số tokens tiêu thụ theo từng user.
  2. Thiết lập hệ thống cảnh báo tự động (Alerting) dựa trên trường giá trị cụ thể, ví dụ: kích hoạt cảnh báo Slack/Email ngay lập tức nếu một log dòng ghi nhận `cost_usd > 0.5` hoặc tỷ lệ sự kiện lỗi `level = error` tăng đột biến trong 5 phút qua.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~148 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Dung lượng chênh lệch (~870 MB) là do phiên bản 1-stage dùng base image `python:3.11` đầy đủ chứa toàn bộ công cụ biên dịch (compilers, build-essential), headers của hệ thống và các file cached tạm khi cài đặt pip. Ngược lại, bản Multi-stage sử dụng stage đầu `builder` để cài dependency, sau đó chỉ copy kết quả thư viện đã build sang stage `runtime` sử dụng `python:3.11-slim` rất gọn nhẹ, loại bỏ hoàn toàn các trình biên dịch không cần thiết khi chạy ứng dụng trên production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa một ký tự trong `app/main.py` rồi build lại:
  - Các layer trước `COPY app/ ./app` (bao gồm base image, `COPY requirements.txt` và `RUN pip install`) được sử dụng lại hoàn toàn từ cache vì file `requirements.txt` không thay đổi.
  - Chỉ có các layer kể từ `COPY app/ ./app` trở đi mới phải chạy lại.
- Nếu đặt `COPY . .` trước `RUN pip install`:
  - Mỗi khi sửa bất kỳ ký tự nào trong source code, Docker sẽ phát hiện thư mục hiện tại thay đổi, từ đó vô hiệu hóa (invalidate) cache từ layer `COPY . .` trở xuống. Kết quả là Docker sẽ phải tải và cài đặt lại toàn bộ các thư viện Python từ đầu qua lệnh `RUN pip install`, làm cho thời gian build image kéo dài thêm nhiều phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện:
  1. Kẻ tấn công phát hiện một lỗ hổng trong code Python (ví dụ: Remote Code Execution - RCE qua việc thực thi lệnh hệ thống tùy ý hoặc leo thang qua đường upload file).
  2. Do container chạy dưới quyền mặc định là `root`, kẻ tấn công thực thi mã độc với đặc quyền tối cao của root trong container.
  3. Kẻ tấn công khai thác tiếp các lỗ hổng nhân hệ điều hành (kernel exploits) hoặc các lỗi cấu hình mount socket Docker (`/var/run/docker.sock`) để thoát khỏi sandbox của container (container escape) và chiếm quyền root trực tiếp trên máy host.
- Lệnh `USER appuser` cắt đứt chuỗi này ngay tại bước 2: Khi ứng dụng chỉ chạy với quyền hạn của một người dùng thường (`appuser`), kẻ tấn công dù có thực thi được mã độc cũng chỉ có đặc quyền hạn chế trong container, không thể ghi vào các thư mục hệ thống quan trọng và không có đủ quyền tối cao để thực hiện leo thang đặc quyền thoát container ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
- Giải thích:
Giả sử cửa sổ đếm theo phút đồng hồ reset vào lúc 10:01:00. Người dùng gửi 10 request dồn dập vào 2 giây cuối của phút trước (từ 10:00:58 đến 10:00:59) - vẫn hợp lệ vì thuộc phút 10:00. Ngay lập tức, họ gửi tiếp 10 request vào 2 giây đầu của phút sau (từ 10:01:00 đến 10:01:01) - vẫn hợp lệ vì thuộc phút 10:01. Tổng cộng hệ thống đã cho phép 20 request đi qua trong vòng 2 giây liên tiếp mà không hề kích hoạt rate limit, tạo ra một đợt burst traffic cực lớn có thể làm sập hệ thống.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Sự khác biệt:
  - **Rate Limit**: Giới hạn số lượng request trong một đơn vị thời gian (tần suất gọi API) để bảo vệ hệ thống khỏi bị quá tải (DDoS).
  - **Cost Guard**: Giới hạn số tiền (chi phí API thực tế phát sinh của LLM) để bảo vệ ví tiền và ngân sách của chủ dịch vụ.
- Tình huống:
  1. *Rate limit cho qua nhưng Cost guard chặn*: User gửi chỉ 1 request trong 1 phút (rất thấp so với rate limit là 10/phút), nhưng request này chứa prompt cực dài và yêu cầu sinh ra kết quả dài (tiêu thụ 100k tokens), chi phí ước tính vượt quá ngân sách tháng còn lại của user đó ➔ Cost guard chặn (HTTP 402), còn Rate limit cho qua.
  2. *Cost guard cho qua nhưng Rate limit chặn*: User gọi API liên tục 30 lần trong vòng 5 giây, mỗi lần chỉ gửi một từ duy nhất (chi phí siêu nhỏ, gần như bằng 0, không ảnh hưởng ngân sách tháng) ➔ Rate limit chặn ngay lập tức (HTTP 429) để bảo vệ server khỏi bị spam nghẽn, còn Cost guard cho qua.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis mất kết nối 30 giây.
2. Endpoint `/health` (lúc này đã bị gộp và kiểm tra Redis) trên cả 3 container đều trả về HTTP 503 (Unhealthy).
3. Hệ thống Orchestrator (Docker Swarm/K8s/Railway) nhận thấy trạng thái Unhealthy của cả 3 container và kích hoạt tiến trình restart cứng (kill và khởi động lại) toàn bộ cả 3 container cùng lúc.
4. Khi 3 container mới đang khởi động lại, chúng tiếp tục kiểm tra Redis và do Redis vẫn đang mất kết nối, chúng lại báo Unhealthy và tiếp tục bị restart vòng lặp (crash loop).
5. Khi Redis kết nối lại thành công sau 30 giây, toàn bộ cụm container đều đang trong trạng thái restart/chưa sẵn sàng hoạt động, dẫn đến toàn bộ hệ thống bị gián đoạn hoàn toàn (downtime diện rộng), biến một sự cố mất kết nối mạng tạm thời thành thảm họa sập hệ thống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử hội thoại được lưu trong một dict Python (RAM cục bộ của từng instance) thay vì Redis, bạn sẽ thấy `history_length` thay đổi ngẫu nhiên không nhất quán sau mỗi lần gọi `/ask`.
- Ví dụ: Load balancer phân phối Request 1 vào Agent A (`history_length = 0` -> lưu vào RAM A). Request 2 được phân phối vào Agent B (`history_length` vẫn trả về `0` vì RAM B chưa có dữ liệu gì). Request 3 phân phối vào Agent A (`history_length = 2`).
Người dùng sẽ cảm thấy AI của bạn bị "mất trí nhớ ngẫu nhiên" sau mỗi lượt chat do trạng thái hội thoại bị chia rẽ giữa các instance khác nhau.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Lỗi gặp phải: Lỗi Health Check Timeout / Crash Loop khi deploy lên cloud.
- Nguyên nhân: Ứng dụng uvicorn được cấu hình mặc định lắng nghe qua cổng cố định `8000` (`--port 8000`), trong khi các nền tảng đám mây như Railway hoặc Render tự động gán một cổng động thông qua biến môi trường `PORT` và yêu cầu container phải bind vào cổng đó để định tuyến traffic từ Internet vào. Do không đọc cổng từ biến môi trường, cloud gọi thử `/health` vào cổng chỉ định và timeout.
- Cách khắc phục: Sửa dòng lệnh chạy uvicorn thành `uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}` để ứng dụng tự động lắng nghe cổng động do platform cung cấp hoặc fallback về 8000 khi chạy local.
