# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Chu Quang Hiếu  Mã học viên: 2A202601344

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu api token có giá trị mặc định, app deploy lên Render vẫn chạy, health check xanh, dashboard báo 'Live' nên k ai biết có vấn đề. Nhưng /chat lúc đó chấp nhận token 'changeme'. Nếu repo này đc public thì sẽ bị lộ key, rủi ro bị ng xấu lợi dụng, thất thoát 1 số tiền lớn. 

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:28:28.402913+00:00", "client_id": "sv-test", "prompt_tokens": 3, "completion_tokens": 37, "usd_cost": 2.265e-05}
> Hai việc print("đã trả lời xong") không làm được:
> 1. Cộng tiền theo client: log là json nên có thể query thẳng đc trên dashboard
> 2. Đặt cảnh báo tự động: lọc event="chat_completed", group theo client_id, sum usd_cost — ra ngay client nào đang đốt tiền nhất trong 24h qua. Với chuỗi văn xuôi thì phải viết regex, và regex vỡ ngay lần đầu ai đó sửa câu chữ trong print.

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
| 1 stage (bản đầu) | 1730 MB (1.73GB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo thật bằng `docker images`: bản 1 stage 1.73GB, bản multi-stage 271MB — nhỏ hơn khoảng 6.4 lần. Chênh lệch ~1.46GB gồm 3 nhóm:
>
> 1. Base image: `python:3.11` bản đầy đủ mang theo gcc, make, header để build, git và bộ công cụ Debian đầy đủ (~1GB). `python:3.11-slim` chỉ giữ Python runtime (~150MB).
> 2. Rác của quá trình cài: pip cache, wheel tạm, file build trung gian. Với multi-stage chúng chết cùng stage `builder`, vì stage runtime chỉ `COPY --from=builder /install /usr/local` — lấy đúng thư viện đã cài xong.
> 3. File không cần copy vào image. Chỗ này em mắc lỗi thật: lần build đầu `.dockerignore` mới chỉ có `.git`, nên `.venv` và cả `.env` bị copy vào. Em kiểm tra bằng `docker run --rm --entrypoint sh day12-chat:prod -c "ls -a /app"` và thấy `/app/.env` nằm trong đó — tức `API_TOKEN` lộ cho bất kỳ ai pull được image, nguy hiểm hơn hẳn vấn đề dung lượng. Sau khi bổ sung `.env`, `.venv`, `__pycache__` vào `.dockerignore` và build lại, image từ 352MB xuống 271MB.
>
> Ý nghĩa thực tế: mỗi lần deploy phải đẩy và kéo image qua mạng, 1.73GB so với 271MB là chênh nhau vài phút mỗi lần — nhân với số lần deploy và rollback lúc có sự cố.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sửa một ký tự trong `app/main.py` rồi build lại, output cho thấy:
>
> ```
> => CACHED [builder 3/4] COPY requirements.txt .
> => CACHED [builder 4/4] RUN pip install --no-cache-dir --prefix=/install -r requirements.txt
> => CACHED [stage-1 2/5] COPY --from=builder /install /usr/local
> => [stage-1 4/5] COPY . .                       0.7s
> => [stage-1 5/5] RUN useradd --create-home ...  0.3s
> ```
>
> Toàn bộ stage `builder` dùng lại cache, kể cả bước `pip install`. Chỉ `COPY . .` và các layer sau nó chạy lại — khoảng 1 giây.
>
> Lý do: Docker băm nội dung file được COPY để quyết định cache. `requirements.txt` không đổi nên layer `pip install` giữ nguyên hash. Layer nằm sau một layer đã đổi thì bắt buộc chạy lại, nên `useradd` chạy lại theo dù nó chẳng liên quan gì tới code.
>
> Nếu đặt `COPY . .` trước `RUN pip install`: sửa một ký tự trong `app/main.py` làm hash của layer COPY đổi, kéo theo `pip install` mất cache và cài lại toàn bộ fastapi, uvicorn, redis, pydantic từ đầu — vài phút thay vì 1 giây, mỗi lần sửa code. Trên CI thì nhân con số đó với mọi commit.
>
> Quy tắc rút ra: xếp lệnh trong Dockerfile theo tần suất thay đổi, thứ ít đổi nhất đặt lên trên.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi container chạy bằng root:
>
> 1. Code Python có lỗ hổng — ví dụ một endpoint truyền input người dùng vào `subprocess`, hoặc một thư viện dính CVE deserialization. Kẻ tấn công thực thi được lệnh tùy ý trong container, với quyền root của container.
> 2. Là root trong container thì ghi được mọi nơi trong filesystem của nó: sửa code app, cài thêm công cụ, đọc mọi biến môi trường — trong đó có `API_TOKEN` và `REDIS_URL` kèm mật khẩu.
> 3. Root trong container và root trên host là cùng một UID 0 (trừ khi bật user namespace remapping, mặc định không bật). Nên nếu container được mount `/var/run/docker.sock`, hoặc chạy `--privileged`, hoặc kernel dính lỗi container escape, thì bước từ root-trong-container sang root-trên-host là chuyện có sẵn công cụ để làm.
> 4. Có root trên host là có mọi container khác trên máy đó, gồm cả Redis và dữ liệu của khách hàng khác.
>
> `USER appuser` cắt ở bước 2. Sau lệnh đó process chạy bằng UID 1000: kẻ tấn công vẫn chạy được lệnh và vẫn đọc được biến môi trường của process, nhưng không ghi được ngoài `/home/appuser`, không cài được package, và bước 3 mất điểm tựa vì UID 1000 trên host chỉ là user thường.
>
> Đây là phòng thủ nhiều lớp chứ không phải bức tường: nó không vá lỗ hổng ở bước 1, chỉ giới hạn thiệt hại. Đổi lại chỉ tốn 2 dòng Dockerfile.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> **Vì sao cần `WWW-Authenticate: Bearer`:** RFC 7235 quy định 401 phải kèm header này. Nó là phần "máy đọc được" của câu trả lời — client nhận 401 và biết ngay server muốn xác thực kiểu gì để thử lại cho đúng. Không có nó, client chỉ biết mình bị từ chối chứ không biết nên gửi Bearer token, Basic auth hay API key ở header nào. Thư viện HTTP, trình duyệt, Postman đều đọc header này. Trong bài em gộp cả ba nhánh lỗi vào một object `HTTPException` dùng chung nên không nhánh nào bị quên header.
>
> **Vì sao cùng một thông báo cho cả ba trường hợp:** vì thông báo chi tiết chính là công cụ dò cho kẻ tấn công. Nếu sai scheme trả "unsupported scheme" còn sai token trả "invalid token", kẻ dò biết được token họ đoán đã đi tới bước so sánh, tức định dạng đúng, chỉ sai giá trị — thông tin miễn phí giúp thu hẹp không gian tìm kiếm.
>
> Cùng lý do form đăng nhập trả "email hoặc mật khẩu không đúng" thay vì "email này không tồn tại": câu thứ hai biến form đăng nhập thành công cụ liệt kê người dùng.
>
> Đánh đổi có thật: người dùng hợp lệ gõ nhầm sẽ khó tự sửa hơn. Chấp nhận được vì đây là API dùng token cấp sẵn, không phải người gõ tay, và server luôn có log để tra chi tiết. Cùng mạch đó, so sánh token phải dùng `secrets.compare_digest` chứ không `==`: `==` dừng ngay tại ký tự đầu tiên khác nhau nên thời gian phản hồi rò rỉ thông tin về số ký tự đầu đã đúng.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> **Có `min(capacity, ...)`: gửi được 10 request.** Xô đầy tối đa là 10 token dù im lặng bao lâu. Sau 10 request liên tiếp (chưa tới 1 giây, chưa kịp nạp thêm gì đáng kể), token còn dưới 1 và request thứ 11 nhận 429 kèm header `Retry-After`.
>
> Em kiểm chứng trên service thật đang chạy ở Render, gọi 15 lần liên tiếp với `BUCKET_CAPACITY=10`:
>
> ```
> 200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
> ```
>
> Đúng 10 rồi chặn.
>
> **Bỏ `min(capacity, ...)`: thành 100 request.** Vì `available()` cộng dồn `(now - last) * refill_per_second` mà không có trần. Im lặng 10 phút ở tốc độ 10 token/phút cho 100 token, và cả 100 tiêu được trong một chớp mắt.
>
> Đó là lỗi nghiêm trọng vì giới hạn mất ý nghĩa: client im lặng 24 giờ tích được 14.400 token và bắn hết trong vài giây — đúng kiểu traffic làm sập backend nhưng lại lọt qua rate limiter một cách "hợp lệ". Mục đích của rate limit là chặn burst; không có trần thì burst càng lớn khi client càng im lặng lâu, ngược hoàn toàn ý định.
>
> `capacity` chính là chỗ khai báo "burst lớn nhất tôi chấp nhận", còn `refill_per_minute` là "tốc độ trung bình dài hạn". Hai tham số tách rời nhau là điểm mạnh của token bucket, và `min()` là dòng giữ cho `capacity` còn ý nghĩa.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> | | $30/tháng | $1/ngày |
> |---|---|---|
> | Thiệt hại tối đa một sự cố | $30 | $1 |
> | Tự hồi phục khi nào | Đầu tháng sau (tối đa 30 ngày) | 00:00 UTC hôm sau (tối đa 24 giờ) |
>
> Sự cố bắt đầu lúc 2h sáng, không ai thức để xử lý.
>
> **Hạn mức tháng:** client đốt liên tục cho tới khi chạm $30 mới bị chặn. Nếu sự cố xảy ra ngày 3 của tháng, em mất trọn $30 và service chết với client đó suốt 27 ngày còn lại, vì hạn mức chỉ reset đầu tháng sau. Tệ cả hai đầu: mất nhiều tiền nhất, đồng thời gián đoạn dài nhất.
>
> **Hạn mức ngày:** đốt tối đa $1 rồi dừng. Sáng ra em thấy log 402 và điều tra lúc tỉnh táo, mất $1 chứ không phải $30. Nếu chưa kịp sửa thì 00:00 UTC key `spend:<client>:<ngày>` đổi sang ngày mới và service tự phục vụ lại, không cần ai can thiệp lúc nửa đêm.
>
> Điểm cốt lõi: cùng một trần chi tiêu, nhưng chia nhỏ theo ngày thì vừa giảm thiệt hại tối đa xuống 1/30, vừa rút thời gian gián đoạn từ hàng tuần xuống dưới một ngày. TTL 3 ngày trên key chi tiêu còn cho phép đối soát lại sau sự cố.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Gộp làm một endpoint có kiểm tra Redis, cụm 3 container, Redis mất kết nối 30 giây:
>
> 1. **t=0s** — Redis mất kết nối. `store.ping()` trả `False` ở cả 3 container cùng lúc, vì chúng dùng chung một Redis.
> 2. **t≈0-10s** — endpoint gộp trả 503. Load balancer rút cả 3 container khỏi vòng phục vụ: mọi request nhận 502/503, kể cả request không cần Redis.
> 3. **t≈10-30s** — orchestrator đọc chính endpoint đó làm liveness probe, thấy fail liên tiếp quá ngưỡng, kết luận "container hỏng" và restart cả 3.
> 4. **t≈30s** — Redis hồi phục. Nhưng 3 container đang khởi động lại: kéo image, chạy startup, chưa nhận traffic.
> 5. **t≈30-60s** — container lên lại, `ping()` thành công, dần được đưa lại vào load balancer.
>
> Kết quả: Redis chập 30 giây biến thành downtime toàn cụm khoảng 60 giây trở lên, cộng nguy cơ crash loop nếu Redis còn phập phù — mỗi lần restart lại fail health check lại restart, và backoff của orchestrator kéo dài thời gian chết thêm.
>
> Tách hai endpoint thì sự cố dừng ở bước 2: `/readyz` trả 503 nên load balancer ngừng gửi traffic, còn `/healthz` vẫn 200 vì process hoàn toàn khỏe mạnh — không có restart nào. Redis hồi phục ở t=30s là `/readyz` xanh lại ngay, traffic quay về sau vài giây.
>
> Câu hỏi mà mỗi probe trả lời là khác nhau: `/healthz` = "có cần restart tôi không?", `/readyz` = "có nên gửi request cho tôi lúc này không?". Restart không sửa được lỗi Redis, nên `/healthz` không được phép biết Redis tồn tại — đó cũng là lý do test của lab bắt `healthz()` không có tham số dependency nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi:** chạy `uvicorn app.main:app --port 8000` để thử container trước khi deploy thì app chết ngay lúc khởi động:
>
> ```
> File "app/lifecycle.py", line 60, in arm
> NotImplementedError: TODO (CP4): cài đặt arm
> ERROR: Application startup failed. Exiting.
> ```
>
> **Tìm nguyên nhân:** đọc ngược traceback, thấy nó xuất phát từ `lifespan` ở `app/main.py:65` — hàm chạy lúc startup có gọi `shutdown_guard.arm()`. Điểm làm em lúng túng ban đầu là `pytest tests/test_cp1.py` vẫn xanh 13/13. Xem `tests/conftest.py` mới hiểu: fixture dựng `TestClient` không dùng `with`, nên lifespan không chạy và test không đụng tới `arm()`. Bài học rút ra là test xanh không có nghĩa app khởi động được — vòng đời startup là đường code mà unit test ở đây cố tình không đi qua, và trên cloud thì đúng đường đó chạy đầu tiên.
>
> **Sửa:** cài đặt `arm()` và `start_draining()` của CP4 (nhớ handler cũ bằng `signal.getsignal` trước rồi mới `signal.signal` ghi đè, và gọi lại handler cũ trong `start_draining`). Sau đó app khởi động bình thường, deploy lên Render và `/healthz` trả 200 ngay lần đầu.
