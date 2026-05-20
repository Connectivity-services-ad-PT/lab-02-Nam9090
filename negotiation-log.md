### Vấn đề 1: Chuẩn hóa giao thức truyền luồng Video
- **Đề xuất**: Luồng thô ban đầu định dùng WebRTC hoặc HLS.
- **Quyết định**: Thống nhất dùng chuẩn RTSP (`rtsp://...`).
- **Lý do**: Phù hợp nhất cho các thư viện xử lý ảnh xử lý AI (như OpenCV) đọc trực tiếp theo thời gian thực.
- **Tác động**: Hệ thống Camera Stream phải trả về đúng định dạng URL này.

### Vấn đề 2: Định dạng dữ liệu ngày tháng
- **Đề xuất**: Dùng chuỗi string tự do hoặc Unix Timestamp.
- **Quyết định**: Sử dụng chuẩn ISO 8601 (`YYYY-MM-DDTHH:mm:ssZ`).
- **Lý do**: OpenAPI hỗ trợ validate tốt kiểu `date-time`.
- **Tác động**: Các trường như `created_at`, `updated_at` tuân thủ nghiêm ngặt định dạng này.

### Vấn đề 3: Xử lý trường hợp Camera không có dữ liệu cập nhật
- **Đề xuất**: Trả về chuỗi rỗng `""`.
- **Quyết định**: Cho phép thuộc tính `updated_at` nhận giá trị `null` (Sử dụng `type: [string, "null"]`).
- **Lý do**: Tuân thủ đúng tư duy OpenAPI 3.1 khi trường đó thực sự chưa có dữ liệu.
- **Tác động**: Phía Client cần kiểm tra nếu `null` thì hiểu là camera chưa từng cập nhật cấu hình.

### Vấn đề 4: Cấu trúc phản hồi khi có lỗi (Error Response)
- **Đề xuất**: Trả về text thuần hoặc object lỗi tự chế `{"error": "message"}`.
- **Quyết định**: Sử dụng chuẩn Problem Details (RFC 7807) với đầy đủ các trường `type`, `title`, `status`, `detail`, `instance`.
- **Lý do**: Đáp ứng tiêu chí chấm điểm tự động của bộ quy tắc `campus-spectral.yaml`.
- **Tác động**: Tất cả các mã lỗi 4xx và 5xx đều phải trả về chung một cấu trúc này.

### Vấn đề 5: Định vị danh tính các biến thể lỗi dữ liệu đầu vào (422)
- **Đề xuất**: Trả về lỗi chung 400 Bad Request.
- **Quyết định**: Tách riêng lỗi 422 Unprocessable Entity kèm theo `oneOf` và `discriminator` để phân loại lỗi phần cứng hoặc lỗi cấu hình sai FPS.
- **Lý do**: Giúp hệ thống phân tích AI biết chính xác nguyên nhân để tự động điều chỉnh cấu hình.
- **Tác động**: Phía viết OpenAPI cần định nghĩa thêm Schemas lỗi đa hình.

### Vấn đề 6: Giới hạn số lượng phiên Stream đồng thời trên một Camera
- **Đề xuất**: Cho phép kết nối vô hạn.
- **Quyết định**: Giới hạn tối đa 3 kết nối đồng thời (`active_connections <= 3`). Nếu vượt quá sẽ trả về lỗi `409 Conflict`.
- **Lý do**: Bảo vệ băng thông phần cứng của hạ tầng Smart Campus khỏi bị treo.
- **Tác động**: Hệ thống AI Vision phải tự ngắt các luồng cũ không dùng tới trước khi mở luồng mới.