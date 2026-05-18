# Phân tích yêu cầu — vai Consumer

- Cặp đàm phán: pair-08-core-analytics-async
- Product: A5 - A6
- Consumer service: Analytics
- Provider service: Core-Business
- Người viết: Bui The Hoang
- Ngày: 18/5/2026

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|policy.decision.created|Theo dõi và thống kê kết quả xử lý luật/chính sách hệ thống để lên biểu đồ tỷ lệ pass/fail.|`decisionId`, `policyId`, `subjectId`, `result`, `timestamp`|`reson`,`correlationId`|

|alert.created|Đếm số lượng cảnh báo thời gian thực, phân loại mức độ nghiêm trọng để hiển thị lên Dashboard.|`alertId`, `type`, `severity`, `timestamp` | `message`, `source_service`, `metadata` ||
|alert.resolved|Ghi nhận thời điểm xử lý xong sự cố để tính toán KPI thời gian phản hồi trung bình (Resolution Time).|`alertId`, `timestamp` | `duration`, `reason`|

---

## 2. API Consumer cần gọi

| Method | Path | Lúc nào gọi? | Kỳ vọng response |

| POST|`/core.events.policy.decision`|Khi Core Business vừa áp dụng một rule/policy nghiệp vụ thành công.|Parse payload JSON, lưu thông tin vào DB Analytics để chạy tiến trình tính toán metric.|
| POST | `/core.events.alert` |  Khi có một cảnh báo mới được tạo lập (alert.created) hoặc được giải quyết xong (alert.resolved).|Kiểm tra loại event (type), cập nhật trạng thái của alert, trigger refresh dữ liệu hiển thị trên dashboard.  |
| POST| `/core.events.health` | Khi Core phát tín hiệu  Heartbeat  | Cập nhật bảng theo dõi uptime/health status của các service  |

---

## 3. Error case Consumer cần xử lý

Tối thiểu 5 case.

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
|1. Lỗi Sai Định Dạng Schema | Message gửi qua Queue không đúng chuẩn JSON, sai kiểu dữ liệu (ví dụ: gửi string thay vì array), hoặc ngày tháng không chuẩn ISO-8601.| Log chi tiết lỗi parse, từ chối nhận (REJECT / NACK) và đẩy thẳng tin nhắn vào Dead Letter Queue (DLQ), không làm nghẽn hàng đợi chính. |


|2. Thiếu thông tin định danh hoặc correlation id|Message hợp lệ JSON nhưng thiếu khóa chính (alertId, decisionId) hoặc thiếu correlationId để truy vết.|Không thể định danh bản ghi hoặc trace luồng. Lập tức loại bỏ event (Drop), báo lỗi cảnh báo hệ thống và đẩy vào DLQ|

|3. Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ|Nhận được dữ liệu đúng cú pháp nhưng sai nghiệp vụ (Ví dụ: Provider gửi status CANCELLED nhưng Analytics chỉ expect RESOLVED hoặc OPEN)|Log warning "Unknown Business State". Lưu tạm dữ liệu dưới nhãn UNMAPPED để bảo toàn số liệu, không làm sập tiến trình xử lý (Crash worker).|

|4. Trùng event/request hoặc retry gây xử lý lặp|Nhận cùng một sự kiện 2 lần (do lỗi mạng, Provider gửi đúp, hoặc Broker tự động retry).|Áp dụng Idempotent: Check Unique ID trong Database trước khi ghi. Nếu đã tồn tại, bỏ qua bước tính toán nghiệp vụ nhưng vẫn trả ACK để xóa tin nhắn khỏi hàng đợi.|

|5. Timeout hoặc lỗi downstream|Database của hệ thống Analytics bị sập/quá tải, hoặc mất kết nối mạng nội bộ khi đang lưu dữ liệu.|Trả tin nhắn lại hàng đợi (NACK kèm tùy chọn requeue=true) để tự động thử lại sau, kết hợp cơ chế ngắt mạch (Circuit Breaker) tạm dừng Consumer nếu lỗi kéo dài.|

| 6. Message Trùng Lặp (Duplicate) | Do cơ chế retry của Provider, cùng một sự kiện bị gửi đến nhiều lần. |Áp dụng Idempotent: Kiểm tra Unique ID (alertId/decisionId) trong DB. Nếu đã tồn tại, bỏ qua xử lý nhưng vẫn trả ACK để Broker xóa tin nhắn. |

| 7. Sự kiện Đến Sai Thứ Tự (Out-of-order) |Event alert.resolved đến trước event alert.created do trễ mạng hoặc xử lý song song. | Kiểm tra DB, nếu chưa thấy bản ghi gốc, đưa message resolved vào trạng thái chờ (Requeue) hoặc lưu tạm để ráp nối sau.|

| 8. 503 | Database của Analytics bị sập. | Trả tin nhắn về lại hàng đợi (NACK với requeue=true) để thử lại sau, hoặc tạm dừng sử dụng message (Pause Consumer). |



---

## 4. Giả định bổ sung

--- 
Giả định 1: Event payload dùng JSON.
Giả định 2: Timestamp dùng ISO-8601 UTC.
Giả định 3: Event có unique iidentifier(enventId hoặc arlertId).
Giả định 4: Queue hỗ trợ retry tối thiểu một lần trong truong hợp consumer mất kết núi.
Giả định 5: Analytics chỉ consume event, không modify dữ liệu Core Business.

---
## 5. Câu hỏi cho Provider

1. 
2. 
3. 

---
## 7. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider đổi kiểu dữ liệu | Consumer parse lỗi | Chốt type/format/pattern |
| Provider thiếu mã lỗi | Consumer khó xử lý lỗi | Chuẩn hóa Problem Details |
