# Phân tích yêu cầu — vai Provider

- Cặp đàm phán: `pair-08-core-analytics-async`
- Product: A
- Provider service: Core Business Service
- Consumer service: Analytics Service
- Người viết:
- Ngày: 

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `CoreEvent` | Event chung do Core Business phát ra cho Analytics xử lý | `eventId`, `eventType`, `eventVersion`, `occurredAt`, `source`, `correlationId`, `data` | `publishedAt`, `metadata` |
| `PolicyDecisionEvent` | Event khi Core Business tạo quyết định nghiệp vụ/policy sau khi xử lý dữ liệu từ IoT, Access Gate hoặc AI Vision | `eventId`, `eventType`, `eventVersion`, `occurredAt`, `source`, `correlationId`, `data.decisionId`, `data.ruleCode`, `data.decision`, `data.targetType`, `data.targetId` | `data.severity`, `data.reason`, `data.actorId`, `data.locationId` |
| `AlertCreatedEvent` | Event khi Core Business tạo cảnh báo bất thường | `eventId`, `eventType`, `eventVersion`, `occurredAt`, `source`, `correlationId`, `data.alertId`, `data.alertType`, `data.severity`, `data.status`, `data.message` | `data.relatedEntityType`, `data.relatedEntityId`, `data.locationId` |
| `AlertResolvedEvent` | Event khi một cảnh báo đã được xử lý hoặc đóng lại | `eventId`, `eventType`, `eventVersion`, `occurredAt`, `source`, `correlationId`, `data.alertId`, `data.resolvedAt`, `data.resolutionStatus` | `data.resolutionNote`, `data.resolvedBy` |
| `Problem` | Mẫu response lỗi dùng chung cho các lỗi 4xx/5xx | `type`, `title`, `status` | `detail`, `instance`, `errors` |

---

## 2. Action/API dự kiến

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| GET | `/health` | Kiểm tra mock server/API contract còn hoạt động | Khi cần kiểm tra môi trường Lab 02, Prism mock hoặc healthcheck |
| POST | `/events/core` | Mô phỏng Analytics nhận một event từ Core Business | Khi Core Business phát sinh `policy.decision.created`, `alert.created`, hoặc `alert.resolved` |
| GET | `/events/core/{eventId}` | Tra cứu một event Core đã được Analytics nhận | Khi cần kiểm tra event đã được ghi nhận hoặc debug tích hợp |
| GET | `/analytics/core/summary` | Lấy thống kê tổng hợp từ các event Core Business | Khi dashboard hoặc tester cần xem tổng số decision, alert, resolved alert |
| GET | `/analytics/core/alerts` | Lấy metric cảnh báo theo loại, severity, trạng thái | Khi dashboard cần hiển thị thống kê cảnh báo chi tiết |

---

## 3. Error case

Tối thiểu 5 case.

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Payload sai định dạng JSON hoặc thiếu cấu trúc event envelope | `Problem` |
| 401 | Thiếu Bearer token khi gọi API yêu cầu xác thực | `Problem` |
| 403 | Token hợp lệ nhưng không có quyền gửi hoặc đọc event Core | `Problem` |
| 404 | Không tìm thấy event theo `eventId` | `Problem` |
| 409 | Event bị gửi trùng `eventId`, Analytics đã xử lý trước đó | `Problem` |
| 422 | Payload đúng JSON nhưng sai nghiệp vụ, ví dụ `eventType` không thuộc danh sách cho phép hoặc `severity` sai enum | `Problem` |
| 500 | Lỗi nội bộ khi Analytics xử lý hoặc lưu event | `Problem` |

---

## 4. Giả định bổ sung

Ghi rõ những điểm user story chưa nói nhưng Provider cần giả định.

- Giả định 1: Core Business sẽ là bên phát event, Analytics là bên nhận event để tổng hợp metric.
- Giả định 2: Cơ chế thật của pair này là async queue/event-based, nhưng trong Lab 02 dùng OpenAPI + Prism để mô phỏng contract qua HTTP.
- Giả định 3: Core Business chỉ gửi các event đã thống nhất gồm `policy.decision.created`, `alert.created`, `alert.resolved`.
- Giả định 4: Tất cả timestamp dùng chuẩn ISO-8601 UTC, ví dụ `2026-05-18T10:15:00Z`.
- Giả định 5: `eventId` là duy nhất để Analytics chống xử lý trùng event khi có retry.
- Giả định 6: Các thay đổi breaking change phải cập nhật `openapi.yaml`, `VERSIONING.md` và được Consumer đồng ý trước khi áp dụng.

---

## 5. Câu hỏi cho Consumer

1. Analytics cần những field nào là bắt buộc để tính dashboard metric: `severity`, `locationId`, `actorId`, `reason` có bắt buộc không?
2. Analytics có cần Core Business gửi cả `publishedAt` hay chỉ cần `occurredAt`?
3. Analytics muốn xử lý event trùng bằng `eventId` hay kết hợp thêm `correlationId`?
4. Với `alert.resolved`, Analytics có cần `resolvedBy` để thống kê người xử lý không?
5. Analytics cần phân loại alert theo `alertType` cố định enum hay cho phép text tự do?
6. Analytics cần giữ dữ liệu event trong bao lâu để phục vụ báo cáo?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Tên field không thống nhất giữa Core và Analytics | Analytics parse lỗi hoặc không tính được metric | Chốt naming trong `openapi.yaml`, dùng cùng schema trong `components/schemas` |
| Core đổi event schema nhưng không báo Analytics | Dashboard sai số liệu hoặc service lỗi khi nhận event mới | Dùng `eventVersion`, ghi thay đổi trong `VERSIONING.md`, cập nhật `changelog` trước khi đổi |
| Event bị gửi trùng do retry hoặc lỗi queue | Analytics đếm trùng số alert/decision | Analytics xử lý idempotency bằng `eventId`; Core phải đảm bảo `eventId` unique |
| Timestamp không thống nhất timezone | Báo cáo theo ngày/giờ bị sai | Bắt buộc dùng ISO-8601 UTC cho `occurredAt`, `publishedAt`, `resolvedAt` |
| `severity` dùng text tự do | Không gom nhóm được cảnh báo theo mức độ | Dùng enum cố định: `low`, `medium`, `high`, `critical` |
| `alert.resolved` thiếu `resolvedAt` | Không tính được thời gian xử lý cảnh báo | Quy định `resolvedAt` là bắt buộc trong `AlertResolvedEvent` |
| Payload lớn hoặc chứa dữ liệu thừa | Mock/test chậm, timeout hoặc khó kiểm thử | Chỉ gửi dữ liệu cần cho Analytics; thống nhất content-type `application/json` và giới hạn kích thước payload |
| Provider chưa sign-off chính thức | Contract có thể chưa được Core chấp nhận | Ghi trạng thái `Draft by Consumer - Pending Provider Review`, yêu cầu Core review/sign-off sau |