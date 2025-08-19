# Node.js Single Thread, Blocking vs Non‑Blocking

Tài liệu này mô tả bản chất **đơn luồng** của Node.js, cách **blocking** làm treo event loop, và cách **non‑blocking** giúp server vẫn phục vụ được nhiều yêu cầu đồng thời. Dưới đây là hướng dẫn chạy demo, kịch bản thử nghiệm, và gợi ý tối ưu.

---

## 1) Tổng quan nhanh
- **Node.js đơn luồng (single‑threaded):** Mọi JavaScript chạy trong **một** luồng chính cùng với **event loop**.
- **Blocking:** Đoạn code chiếm CPU lâu (ví dụ vòng lặp bận) sẽ **chặn** event loop → tất cả request khác **đợi**.
- **Non‑blocking:** Đưa tác vụ chờ I/O hoặc delay sang cơ chế bất đồng bộ (callback/timer/promise/async) → event loop rảnh để xử lý request khác.

---

## 2) Mã demo
```js
const http = require('http')
function wait(millisec) {
    var now = new Date;
    while (new Date - now <= millisec) ;
}
http.createServer((req, res)=> {
    if (req.url === '/') {
        res.writeHead(200, {"Content-Type": 'text/html'});
        res.write('hello')
        res.end()
    }
    if (req.url === '/wait') {
        wait(5000)
        console.log('wait');
        res.writeHead(200, {"Content-Type": 'text/html'});
        res.write('Done>>>wait')
        res.end()
    }
    if (req.url === '/timeout') {
        setTimeout(()=> {
            res.writeHead(200, {"Content-Type": 'text/html'});
            res.write('Done>>>timeout')
            res.end()
        }, 5000)
        console.log('timeout');
    }
}).listen(3000, "127.0.0.1", function() {
    console.log('server start at http://127.0.0.1:3000')
})
```

**Ý nghĩa các route**
- `/` → phản hồi ngay `hello`.
- `/wait` → **blocking 5s** bằng vòng lặp bận (busy‑wait). Trong lúc này các request khác bị **treo**.
- `/timeout` → **non‑blocking 5s** bằng `setTimeout`. Trong thời gian chờ, server vẫn xử lý được request khác.

---

## 3) Cách chạy nhanh
### Yêu cầu
- Node.js >= 14

### Bước chạy
```bash
# 1) Lưu file thành server.js
# 2) Chạy
node server.js
# 3) Mở trình duyệt
#    http://127.0.0.1:3000
```
Nếu port bận (EADDRINUSE), đổi port trong `listen` (vd: 3001) hoặc kill process đang chiếm port.

---

## 4) Kịch bản thử nghiệm (thấy rõ blocking vs non‑blocking)
### 4.1. Kiểm tra từng route
```bash
# A) Route nhanh
curl -i http://127.0.0.1:3000/

# B) Route blocking 5s
curl -i http://127.0.0.1:3000/wait

# C) Route non‑blocking 5s
curl -i http://127.0.0.1:3000/timeout
```
**Kết quả mong đợi**
- `/` trả lời ngay: `hello`.
- `/wait` mất ~5s mới trả lời `Done>>>wait`. Trong ~5s này, thử gọi `/` ở cửa sổ khác sẽ **đợi** xong `/wait` mới ra.
- `/timeout` trả về sau ~5s, nhưng trong lúc đợi bạn vẫn gọi `/` được **ngay**.

> Mẹo: Mở 2 tab terminal. Tab 1 gọi `/wait`, ngay lập tức ở tab 2 gọi `/` → sẽ bị treo đến khi `/wait` xong.

### 4.2. Mô phỏng nhiều request đồng thời
Dùng `autocannon` (hoặc `wrk`) để thấy độ nghẽn:
```bash
npx autocannon -c 20 -d 10 http://127.0.0.1:3000/wait     # blocking: RPS rất thấp
npx autocannon -c 20 -d 10 http://127.0.0.1:3000/timeout  # non‑blocking: RPS cao hơn rõ
```

---

## 5) Vì sao `/wait` làm treo server?
- Hàm `wait` là **CPU‑bound** và chạy ngay trên **main thread**.
- Event loop không có cơ hội chuyển sang xử lý callback/request khác cho đến khi vòng lặp kết thúc.
- Kết quả: mọi request đến trong khoảng đó bị **xếp hàng** và **tăng độ trễ**.

**Tóm tắt event loop ngắn gọn**
1. Nhận event (I/O, timer, request HTTP).
2. Xử lý JS hiện tại.
3. Nếu JS đang chạy là blocking (CPU bận), event loop **không** quay được bước tiếp theo.
4. Non‑blocking (timer/I/O) đặt callback cho tương lai, thả event loop rảnh ngay.

---

## 6) Cách làm đúng: tránh blocking
### 6.1. Không dùng busy‑wait để delay
- Thay bằng timer hoặc promise:
```js
const sleep = ms => new Promise(r => setTimeout(r, ms));

http.createServer(async (req, res) => {
  if (req.url === '/sleep') {
    await sleep(5000); // non‑blocking delay
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Done>>>sleep');
  }
}).listen(3000);
```

### 6.2. Với tác vụ CPU nặng (hash, nén, ML, parse lớn)
- **worker_threads** hoặc **child_process** để tách CPU‑bound khỏi main thread.
- Hoặc scale ngang bằng **cluster**/PM2/K8s + nhiều instance.

### 6.3. Với I/O (DB, file, HTTP)
- Luôn dùng API **async** (callback/promises/async‑await).
- Tránh thư viện sync (vd: `fs.readFileSync` trong request).

---

## 7) Mẫu refactor code demo (giữ hành vi nhưng non‑blocking)
```js
const http = require('http');
const sleep = ms => new Promise(r => setTimeout(r, ms));

http.createServer(async (req, res) => {
  if (req.url === '/') {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    return res.end('hello');
  }

  if (req.url === '/wait') {
    // Thay busy‑wait bằng sleep non‑blocking
    await sleep(5000);
    console.log('wait');
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    return res.end('Done>>>wait');
  }

  if (req.url === '/timeout') {
    setTimeout(() => {
      res.writeHead(200, { 'Content-Type': 'text/plain' });
      res.end('Done>>>timeout');
    }, 5000);
    console.log('timeout');
    return; // Kết thúc handler, callback sẽ tự gửi response
  }

  res.writeHead(404);
  res.end('Not Found');
}).listen(3000, '127.0.0.1', () => {
  console.log('server start at http://127.0.0.1:3000');
});
```

---

## 8) Gỡ lỗi & lưu ý
- **EADDRINUSE:** Port bận → đổi port hoặc kill process.
- **Headers already sent:** Không `res.end` nhiều lần hoặc ghi header sau khi đã `end`.
- **Treo toàn bộ app khi 1 request nặng:** Dấu hiệu điển hình của blocking → kiểm tra CPU, tìm đoạn sync/busy‑wait.
- **Timeout phía client:** Khi server bị nghẽn do blocking, client dễ gặp timeout.

---

## 9) Checklist nhanh cho dự án thực tế
- [ ] Không dùng busy‑wait.
- [ ] Tránh API sync trong path nóng.
- [ ] Di chuyển CPU‑bound sang worker/queue.
- [ ] Dùng async/await cho I/O.
- [ ] Giới hạn concurrency (pool DB/HTTP).
- [ ] Giám sát RPS, latency, event loop delay.

---

## 10) FAQ
**Q:** Node.js đơn luồng thì làm sao xử lý nhiều kết nối?

**A:** Bản thân I/O là non‑blocking, event loop chuyển qua lại giữa nhiều socket nhanh chóng. Vấn đề chỉ xảy ra khi bạn tự chặn event loop bằng tác vụ CPU sync.

**Q:** Khi nào nên dùng worker_threads?

**A:** Khi tác vụ CPU > vài chục ms và lặp lại thường xuyên (hash, parse, transform) → tách sang worker để không chặn event loop.

---

## 11) Tài liệu tham khảo đề xuất
- Tìm hiểu sâu về **Event Loop**, **Timers**, **Poll**, **Check**, **Microtasks** trong Node.js docs.
- Khái niệm **libuv** và cách Node xử lý I/O đa nền tảng.

---

## 12) Gợi ý bài tập tự luyện
1) Viết route `/cpu` tính số Fibonacci lớn theo 2 cách: sync vs worker_threads. So sánh RPS.
2) Chuyển toàn bộ route sync trong dự án hiện tại sang async/await, đo **event loop delay**.
3) Thử thêm log thời gian xử lý (ms) cho từng request để phát hiện chỗ nghẽn.

