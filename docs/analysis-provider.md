# Phân tích yêu cầu — vai Provider

- Cặp đàm phán: Pair 01 (hoặc tùy theo docs/pairing-matrix.md)
- Product: Product A
- Provider service: Camera Stream Service (Dịch vụ quản lý và truyền luồng Camera)
- Consumer service: AI Vision / Analytics Service (Dịch vụ phân tích hình ảnh AI)
- Người viết: Trần Đình Nam (Nam9090)
- Ngày: 20-05-2026

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `Camera` | Đối tượng quản lý thông tin phần cứng và trạng thái của Camera trong khu vực campus. | `id` (string), `name` (string), `location` (string), `status` (string) | `resolution` (string), `fps` (integer), `updated_at` (string, định dạng date-time hoặc `null`) |
| `StreamSession` | Phiên kết nối và phân phối luồng dữ liệu (RTSP/HLS) từ Camera để Consumer lấy dữ liệu xử lý. | `session_id` (string), `camera_id` (string), `stream_url` (string), `created_at` (string) | `expires_at` (string hoặc `null`), `active_connections` (integer) |

---

## 2. Action/API dự kiến

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| GET | `/cameras` | Lấy danh sách toàn bộ Camera đang hoạt động trong hệ thống. | Khi Consumer muốn quét và tìm mã ID của các Camera có sẵn trong khu vực cần giám sát. |
| GET | `/cameras/{id}` | Lấy chi tiết thông tin và cấu hình hiện tại của một Camera cụ thể. | Khi Consumer cần kiểm tra độ phân giải, số khung hình (FPS) trước khi xử lý hình ảnh. |
| POST | `/cameras/{id}/stream/start` | Yêu cầu khởi tạo hoặc lấy luồng dữ liệu Live Stream (URL RTSP/HLS) của Camera. | Khi bên AI Vision bắt đầu kích hoạt một tiến trình phân tích luồng video (nhận diện khuôn mặt, biển số...). |
| POST | `/cameras/{id}/stream/stop` | Yêu cầu hủy bỏ phiên truyền luồng dữ liệu của Camera để giải phóng băng thông. | Khi hệ thống phân tích AI dừng luồng dữ liệu hoặc camera đó không cần giám sát nữa. |

---

## 3. Error case

Cấu trúc Response lỗi tuân thủ nghiêm ngặt theo chuẩn **Problem Details (RFC 7807)**.

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Payload gửi lên sai định dạng hoặc thiếu trường bắt buộc khi khởi tạo phiên Stream. | `{"type": "https://example.com/probs/bad-request", "title": "Bad Request", "status": 400, "detail": "Missing required field: camera_id", "instance": "/cameras/invalid-id/stream/start"}` |
| 401 | Consumer gọi API nhưng không kèm theo Bearer token hoặc token đã hết hạn. | `{"type": "https://example.com/probs/unauthorized", "title": "Unauthorized", "status": 401, "detail": "Full authentication is required to access this resource.", "instance": "/cameras"}` |
| 403 | Token hợp lệ nhưng tài khoản/Service của Consumer không được cấp quyền xem Camera này. | `{"type": "https://example.com/probs/forbidden", "title": "Forbidden", "status": 403, "detail": "You do not have permission to view streams from Zone C.", "instance": "/cameras/cam-102/stream/start"}` |
| 404 | Mã `camera_id` truyền vào trên URL không tồn tại trong cơ sở dữ liệu Smart Campus. | `{"type": "https://example.com/probs/not-found", "title": "Resource Not Found", "status": 404, "detail": "Camera with ID cam-9999 does not exist.", "instance": "/cameras/cam-9999"}` |
| 409 | Camera đang bị khóa bảo trì phần cứng hoặc luồng stream đã đạt giới hạn kết nối tối đa. | `{"type": "https://example.com/probs/conflict", "title": "Conflict", "status": 409, "detail": "Camera stream is currently locked for maintenance.", "instance": "/cameras/cam-102/stream/start"}` |
| 422 | Dữ liệu đúng định dạng JSON nhưng giá trị cấu hình FPS hoặc Resolution vượt giới hạn Camera cho phép. | `{"type": "https://example.com/probs/unprocessable-entity", "title": "Unprocessable Entity", "status": 422, "detail": "Requested FPS (120) exceeds hardware limit (60).", "instance": "/cameras/cam-102"}` |

---

## 4. Giả định bổ sung

- Giả định 1: Mỗi Camera chỉ cho phép tối đa 3 kết nối Stream đồng thời (`active_connections <= 3`) để tránh quá tải băng thông phần cứng Camera gốc. Nếu vượt quá, hệ thống sẽ trả về lỗi `409 Conflict`.
- Giả định 2: Định dạng URL luồng dữ liệu (`stream_url`) mặc định trả về sẽ là RTSP cho các tác vụ AI phân tích nội bộ, tuy nhiên có hỗ trợ HLS nếu tiêu đề Request yêu cầu cụ thể.
- Giả định 3: Thuộc tính thời gian kết thúc phiên `expires_at` có thể mang giá trị `null` đối với các Camera giám sát an ninh trọng điểm cần duy trì luồng truyền tải 24/7 không bao giờ hết hạn.

---

## 5. Câu hỏi cho Consumer

1. Bên dịch vụ AI Vision (Consumer) có thể tự động xử lý việc kết nối lại (Re-connect) luồng dữ liệu khi hạ tầng mạng gặp sự cố mất kết nối đột ngột hay cần Provider gửi sự kiện cảnh báo?
2. Consumer có yêu cầu giới hạn thời gian (TTL) cho một phiên Live Stream hay không, hay phiên sẽ chạy vô hạn cho đến khi có lệnh `POST .../stream/stop`?
3. Định dạng phân giải video mong muốn tối ưu cho các mô hình AI của bạn là gì (1080p, 720p, hay luồng thô Sub-stream)?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Lệch chuẩn định dạng URL luồng dữ liệu | Consumer không thể parse hoặc không kết nối được vào trình phát video độc lập. | Thống nhất định dạng URL trong tài liệu `openapi.yaml` (ví dụ: `rtsp://username:password@ip:port/stream`). |
| Độ trễ mạng (Latency) cao khi truyền tải dữ liệu dung lượng lớn | Mô hình AI xử lý nhận diện hành vi bị chậm trễ so với thời gian thực. | Provider cam kết tối ưu hóa bộ đệm đóng gói và giới hạn kích thước gói video phân phối (Size limit). |
| Rò rỉ thông tin luồng Camera bảo mật | Các khu vực hạn chế trong Campus bị truy cập trái phép. | Bắt buộc sử dụng mã hóa token bảo mật Bearer JWT riêng biệt cho từng vùng Camera chỉ định và kiểm tra kỹ quyền ở bước gateway. |