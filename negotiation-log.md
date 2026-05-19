# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: `pair-08-core-analytics-async`
- Product: A
- Provider: A6 Core Business Service
- Consumer: A5 Analytics Service
- Phiên: v1.0
- Ngày: 2026-05-19

---

## Issue #1

- Raised by: Consumer
- Endpoint: `POST /events/core`
- Concern: Analytics cần biết chính xác Core Business sẽ gửi những loại event nào để phân loại và xử lý dữ liệu.
- Proposal: Thống nhất 3 loại event chính:
  - `policy.decision.created`
  - `alert.created`
  - `alert.resolved`
- Resolution: Modified
- Rationale: Vì Core Business chưa sign-off chính thức nên 3 event trên được chốt ở mức draft v1.0 từ phía Consumer. Sau khi Provider review, nếu cần có thể bổ sung event khác.
- Impact: Analytics có thể thiết kế schema, mock request và dashboard metric dựa trên 3 loại event này.

---

## Issue #2

- Raised by: Consumer
- Endpoint: `POST /events/core`
- Concern: Nếu mỗi event có cấu trúc khác nhau hoàn toàn, Analytics sẽ khó validate, trace và chống trùng dữ liệu.
- Proposal: Mọi event từ Core Business phải dùng chung event envelope gồm:
  - `eventId`
  - `eventType`
  - `eventVersion`
  - `occurredAt`
  - `publishedAt`
  - `source`
  - `correlationId`
  - `data`
- Resolution: Accepted
- Rationale: Event envelope giúp Analytics xử lý nhiều loại event theo một chuẩn chung, đồng thời hỗ trợ trace log và idempotency.
- Impact: Core Business cần đóng gói payload theo đúng cấu trúc envelope trước khi gửi sang Analytics.

---

## Issue #3

- Raised by: Consumer
- Endpoint: `POST /events/core`
- Concern: Trong cơ chế async queue, event có thể bị gửi lại nhiều lần do retry hoặc lỗi mạng, làm Analytics đếm trùng số liệu.
- Proposal: Core Business phải đảm bảo `eventId` là duy nhất cho mỗi event. Analytics sẽ dùng `eventId` để kiểm tra trùng lặp.
- Resolution: Accepted
- Rationale: `eventId` unique là điều kiện cần để Analytics xử lý idempotency và tránh sai số dashboard.
- Impact: Nếu Core gửi trùng `eventId`, Analytics có thể trả lỗi `409 Conflict` hoặc bỏ qua event đã xử lý.

---

## Issue #4

- Raised by: Consumer
- Endpoint: `POST /events/core`, `GET /analytics/core/summary`
- Concern: Analytics cần tổng hợp dữ liệu theo ngày, giờ và khoảng thời gian. Nếu timestamp không thống nhất timezone thì báo cáo có thể sai.
- Proposal: Tất cả timestamp dùng chuẩn ISO-8601 UTC, ví dụ:
  - `2026-05-18T10:15:00Z`
- Resolution: Accepted
- Rationale: ISO-8601 UTC giúp tránh sai lệch múi giờ giữa các service và dễ lọc dữ liệu theo khoảng thời gian.
- Impact: Các field như `occurredAt`, `publishedAt`, `resolvedAt` phải dùng cùng chuẩn thời gian.

---

## Issue #5

- Raised by: Consumer
- Endpoint: `POST /events/core`, `GET /analytics/core/alerts`
- Concern: Nếu `severity` dùng text tự do, Analytics khó gom nhóm cảnh báo theo mức độ.
- Proposal: Dùng enum cố định cho `severity`:
  - `low`
  - `medium`
  - `high`
  - `critical`
- Resolution: Accepted
- Rationale: Enum giúp dữ liệu nhất quán và dashboard có thể lọc/thống kê chính xác theo mức độ cảnh báo.
- Impact: Core Business phải gửi `severity` theo đúng danh sách enum. Nếu gửi giá trị khác, Analytics có thể trả `422 Unprocessable Entity`.

---

## Issue #6

- Raised by: Consumer
- Endpoint: `POST /events/core`
- Concern: Analytics cần biết khi nào alert được xử lý để tính số cảnh báo đã giải quyết và thời gian xử lý trung bình.
- Proposal: Event `alert.resolved` cần có các field:
  - `alertId`
  - `resolvedAt`
  - `resolutionStatus`
  - `resolutionNote`
  - `resolvedBy`
- Resolution: Modified
- Rationale: `alertId`, `resolvedAt`, `resolutionStatus` là bắt buộc. `resolutionNote` và `resolvedBy` có thể null nếu Core Business chưa có dữ liệu.
- Impact: Analytics có thể tính resolved alert count, open/resolved ratio và average resolution time.

---

# Chốt hợp đồng v1.0

Provider sign-off: Pending — A6 Core Business
Consumer sign-off: Accepted — A5 Analytics 
Witness (GV/TA): Pending  
Date: 2026-05-19

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|