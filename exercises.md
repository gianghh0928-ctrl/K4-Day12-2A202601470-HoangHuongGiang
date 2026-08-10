# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời ngay bên dưới từng câu hỏi.

> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hoàng Hương Giang  Mã học viên: 2A202601470

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên Cloud (Render/Railway), nếu ta quên cài đặt biến môi trường `API_TOKEN`. Nếu `api_token` có mặc định là `"changeme"`, ứng dụng vẫn khởi động và nhận request. Kẻ tấn công hoặc bot quét Internet có thể dò ra token mặc định `"changeme"` và gọi API `/chat` miễn phí, tiêu tốn toàn bộ ngân sách LLM của bạn trước khi bạn kịp phát hiện. Ngược lại, việc "chết sớm" làm ứng dụng báo lỗi `ValidationError` ngay lúc deploy, giúp ta phát hiện và bổ sung secret ngay lập tức trước khi mở dịch vụ ra công khai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:58:00.123456+00:00", "client_id": "sv-test", "prompt_tokens": 3, "completion_tokens": 37, "usd_cost": 0.00002265}
```

Hai việc làm được:
1. **Truy vấn & Lọc tự động (Query & Filtering):** Các hệ thống gom log (Cloud Logging, Datadog) có thể tự parse JSON để truy vấn lọc theo thuộc tính (ví dụ: lọc tất cả log có `severity == "ERROR"` hoặc theo `client_id`), điều mà log văn bản thuần từ `print()` không thể làm được.
2. **Thống kê & Cảnh báo chỉ số (Aggregation & Alerting):** Dễ dàng tính tổng chi phí `usd_cost` theo từng client trong ngày hoặc theo dõi lượng token đã dùng để vẽ biểu đồ và kích hoạt cảnh báo tự động khi vượt hạn mức.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.8 GB |
| Multi-stage | ~300 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~1.5 GB) bao gồm các công cụ biên dịch (compilers như `gcc`, `g++`, `make`), bộ thư viện phát triển (`build-essential`, header files), bộ nhớ tạm trong quá trình `pip install` (cache wheel, build artifacts) và Python SDK bản đầy đủ không cần thiết cho quá trình chạy. Ở bản Multi-stage, Stage `builder` biên dịch dependencies rồi bị loại bỏ hoàn toàn, chỉ copy sản phẩm chạy cuối cùng sang Stage `runtime` siêu nhẹ (`python:3.11-slim`).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Các layer từ đầu cho đến bước `RUN pip install ...` đều được giữ nguyên và dùng lại từ cache (vì `requirements.txt` không đổi). Chỉ các layer từ bước `COPY app ./app` trở về sau mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mọi lần sửa code (dù chỉ 1 ký tự trong `main.py`) đều làm thay đổi checksum của layer `COPY . .`, khiến Docker hủy cache từ layer đó trở đi. Kết quả là Docker phải tải và cài lại toàn bộ thư viện ở mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện khi chạy root:
  1. Kẻ tấn công khai thác lỗ hổng RCE trong code Python để thực thi lệnh shell bên trong container.
  2. Do container chạy với quyền root (UID 0), lệnh shell này có đầy đủ quyền root trong container.
  3. Kẻ tấn công khai thác tiếp lỗ hổng container breakout hoặc truy cập các docker socket/volume mount để chiếm quyền root trực tiếp trên máy host.
- Lệnh `USER` cắt đứt chuỗi ở bước 2: Lệnh `USER appuser` (UID 10001) hạ quyền tiến trình xuống user thường không có đặc quyền. Khi kẻ tấn công thực thi được lệnh bên trong container, họ bị giới hạn bởi quyền của `appuser`, không thể đọc/ghi file hệ thống nhạy cảm hay thực hiện các thao tác nâng quyền root trên host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

- **Kèm header `WWW-Authenticate: Bearer`:** Đây là quy định bắt buộc theo chuẩn HTTP (RFC 6750) đối với mã lỗi 401 Unauthorized để thông báo chuẩn hóa cho client/trình duyệt biết phương thức xác thực mà API yêu cầu (Bearer Token).
- **Trả cùng một thông báo lỗi:** Theo nguyên tắc an ninh "Least Information Disclosure", việc trả cùng một thông báo lỗi chung ngăn việc tiết lộ thông tin nội bộ cho kẻ tấn công. Nếu trả về lý do cụ thể (như "sai scheme" hay "sai token"), kẻ tấn công sẽ biết từng bước dò để mò ra cấu trúc header và bruteforce token.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

- Với cấu hình hiện tại: Client gửi được **10 request** liên tiếp trước khi bị lỗi 429 (vì xô chỉ tích tối đa bằng `capacity = 10` token).
- Nếu bỏ đoạn `min(capacity, ...)`: Con số đó sẽ thành **110 request** (10 token ban đầu + 10 phút x 10 token/phút = 100 token nạp thêm). Lý do: Thiếu `min(capacity, ...)`, xô sẽ nạp token vô hạn theo thời gian, cho phép client "bùng nổ" bắn hàng trăm request liên tiếp trong 1 giây, phá hỏng mục đích Rate Limiting.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- **Hạn mức $30/tháng:** Thiệt hại tối đa trong sự cố 2h sáng có thể đốt hết toàn bộ **$30** chỉ trong vài giờ. Service sẽ bị khóa trong suốt phần còn lại của tháng và chỉ tự hồi phục vào đầu tháng sau (trừ khi reset thủ công).
- **Hạn mức $1/ngày:** Thiệt hại tối đa cho sự cố 2h sáng chỉ gói gọn trong **$1** (bằng 1/30 hạn mức tháng). Service sẽ tự động hồi phục và cho phép gọi lại vào 00:00 UTC sáng hôm sau khi sang ngày mới mà không cần con người can thiệp.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis gặp sự cố mất kết nối trong 30 giây.
2. Endpoint gộp kiểm tra Redis trả về 503 cho cả 3 container.
3. Orchestrator (Docker/K8s) hiểu nhầm liveness probe hỏng ➔ Tiến hành **kill và restart lại toàn bộ 3 container** cùng lúc.
4. Khi Redis hồi phục sau 30 giây, các container vẫn đang trong quá trình boot up và chưa sẵn sàng, khiến ứng dụng bị downtime hoàn toàn (sự cố nhỏ của Redis biến thành sự cố toàn hệ thống).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:** `ValidationError: 1 validation error for Settings / api_token / Field required` khiến container dừng đột ngột ngay sau khi deploy trên Render (`Create web service day12-chat deploy failed`).
- **Cách tìm nguyên nhân:** Mở tab **Logs / Events** của service `day12-chat` trên Render Dashboard và thấy traceback khởi động ứng dụng bị ném ngoại lệ do thiếu biến `API_TOKEN`.
- **Cách sửa:** Vào tab **Environment** trên Render Dashboard, bấm **Add Environment Variable**, điền key `API_TOKEN` với giá trị token bí mật, bấm **Save Changes** rồi chọn **Manual Deploy** lại.

