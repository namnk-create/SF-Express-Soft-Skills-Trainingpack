# Chuyển từ Firebase Firestore sang Cloudflare (Worker + D1)

Bộ file:
- `index.html` — file sổ tay đào tạo, đã bỏ Firebase SDK, gọi API qua `fetch()`.
- `worker.js` — Cloudflare Worker: 2 endpoint `POST /api/submit` và `GET /api/stats`.
- `schema.sql` — schema D1 (2 bảng `quiz_submissions`, `quiz_stats_public`).
- `wrangler.toml` — cấu hình deploy Worker + binding D1.

## Triển khai lần đầu

```bash
npm install -g wrangler        # nếu chưa có
wrangler login

# 1. Tạo D1 database
wrangler d1 create sfexpress-quiz
# → copy "database_id" từ kết quả, dán vào wrangler.toml

# 2. Chạy schema lên D1 (nhớ --remote để chạy trên Cloudflare, không phải máy local)
wrangler d1 execute sfexpress-quiz --remote --file=./schema.sql

# 3. Deploy Worker
wrangler deploy
# → copy URL vừa in ra, dạng https://sfexpress-quiz-worker.<subdomain>.workers.dev
```

## Trỏ index.html vào Worker

Mở `index.html`, tìm hằng số `WORKER_BASE_URL` (gần đầu file, trong khối
`<!-- ===== CLOUDFLARE (nộp bài qua Worker + D1) ===== -->`), dán URL Worker vừa
deploy vào (không có dấu `/` ở cuối):

```js
const WORKER_BASE_URL = "https://sfexpress-quiz-worker.<subdomain>.workers.dev";
```

Sau đó có thể deploy `index.html` lên bất kỳ nơi lưu trữ tĩnh nào (Cloudflare Pages,
GitHub Pages, S3, v.v.) — không còn phụ thuộc Firebase Hosting nữa.

## Xem dữ liệu đầy đủ (thay cho việc vào Firebase Console)

```bash
wrangler d1 execute sfexpress-quiz --remote \
  --command="SELECT * FROM quiz_submissions ORDER BY submitted_at DESC LIMIT 50"
```

Hoặc vào Cloudflare Dashboard → Workers & Pages → D1 → chọn database
`sfexpress-quiz` → tab Console để chạy SQL trực tiếp trên trình duyệt.

## Thêm chương trình đào tạo mới (nhân bản file)

Giống hệt quy trình cũ với Firestore, chỉ khác tên biến:
1. Copy `index.html` thành file mới.
2. Đổi `PROGRAM_ID` sang hậu tố mới (VD: `"_giao-nhan"`) — **không đổi** `PROGRAM_ID`
   của chương trình gốc (giữ `""`).
3. `WORKER_BASE_URL` giữ nguyên — mọi chương trình dùng chung 1 Worker + 1 D1
   database, dữ liệu tách biệt nhau nhờ cột `program_id`.
4. Không cần sửa gì trong `worker.js` hay `schema.sql`.

## CORS

Mặc định `worker.js` cho phép mọi origin gọi API (`ALLOWED_ORIGIN = "*"`). Nếu muốn
giới hạn chỉ domain lưu trữ `index.html` được gọi, sửa hằng số này trong `worker.js`
thành domain thật rồi `wrangler deploy` lại.
