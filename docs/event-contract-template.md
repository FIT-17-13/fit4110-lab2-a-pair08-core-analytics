# Event Contract sơ bộ — dùng cho dependency Queue async

> File này chỉ dùng cho các cặp Queue async ở Lab 02 để ghi nhận thỏa thuận ban đầu. Đặc tả chi tiết bằng AsyncAPI sẽ chuyển sang Lab 03.

## 1. Thông tin dependency

- Dependency số: `pair-08-core-analytics-async`
- Producer: A6 Core Business Service
- Consumer: A5 Analytics Service
- Cơ chế: Queue async
- Event/topic dự kiến:
  - `core.policy.decision.created`
  - `core.alert.created`
  - `core.alert.resolved`
- Người ghi: A5 Analytics
- Ngày: 2026-05-19

---

## 2. Mục đích nghiệp vụ

Core Business là service xử lý luật nghiệp vụ trung tâm. Service này nhận dữ liệu từ IoT Ingestion, Access Gate và AI Vision, sau đó đưa ra quyết định nghiệp vụ hoặc tạo cảnh báo nếu phát hiện bất thường.

Analytics là service tổng hợp và phân tích dữ liệu. Analytics cần nhận event từ Core Business để:

- thống kê số quyết định policy;
- thống kê số cảnh báo;
- thống kê cảnh báo theo mức độ nghiêm trọng;
- thống kê cảnh báo đã xử lý/chưa xử lý;
- phục vụ dashboard và báo cáo vận hành.

Dependency này dùng cơ chế Queue async. Trong Lab 02, nhóm chỉ ghi nhận event contract sơ bộ. Đặc tả topic, retry, dead-letter queue và consumer group sẽ được làm rõ hơn ở Lab 03.

---

## 3. Event name / topic

| Mục | Giá trị |
|---|---|
| Event name | `policy.decision.created` |
| Topic/queue | `core.policy.decision.created` |
| Producer | `core-business` |
| Consumer | `analytics` |

| Mục | Giá trị |
|---|---|
| Event name | `alert.created` |
| Topic/queue | `core.alert.created` |
| Producer | `core-business` |
| Consumer | `analytics` |

| Mục | Giá trị |
|---|---|
| Event name | `alert.resolved` |
| Topic/queue | `core.alert.resolved` |
| Producer | `core-business` |
| Consumer | `analytics` |

---

## 4. Payload tối thiểu

### 4.1. Event envelope chung

Tất cả event Core Business gửi sang Analytics cần có envelope chung:

```json
{
  "eventId": "EVT-2026-0001",
  "eventType": "policy.decision.created",
  "eventVersion": "1.0.0",
  "occurredAt": "2026-05-19T10:00:00Z",
  "correlationId": "CORR-2026-0001",
  "source": "core-business",
  "data": {}
}

```
### 4.2. policy.decision.created
```json
{
  "eventId": "EVT-2026-0001",
  "eventType": "policy.decision.created",
  "eventVersion": "1.0.0",
  "occurredAt": "2026-05-19T10:00:00Z",
  "correlationId": "CORR-2026-0001",
  "source": "core-business",
  "data": {
    "decisionId": "DEC-2026-0001",
    "ruleCode": "ACCESS_AFTER_HOURS",
    "decision": "deny",
    "targetType": "access_event",
    "targetId": "ACC-2026-0001",
    "severity": "high",
    "reason": "Access outside allowed time"
  }
}
```
### 4.3. alert.created
```json
{
  "eventId": "EVT-2026-0002",
  "eventType": "alert.created",
  "eventVersion": "1.0.0",
  "occurredAt": "2026-05-19T10:05:00Z",
  "correlationId": "CORR-2026-0002",
  "source": "core-business",
  "data": {
    "alertId": "ALT-2026-0001",
    "alertType": "unauthorized_access",
    "severity": "high",
    "status": "open",
    "message": "Unauthorized access detected",
    "relatedEntityType": "access_event",
    "relatedEntityId": "ACC-2026-0001"
  }
}
```
### 4.4. alert.resolved
```json
{
  "eventId": "EVT-2026-0003",
  "eventType": "alert.resolved",
  "eventVersion": "1.0.0",
  "occurredAt": "2026-05-19T10:30:00Z",
  "correlationId": "CORR-2026-0003",
  "source": "core-business",
  "data": {
    "alertId": "ALT-2026-0001",
    "resolvedAt": "2026-05-19T10:30:00Z",
    "resolutionStatus": "resolved",
    "resolutionNote": "Security team handled the alert"
  }
}
```
| Vấn đề                              | Quyết định tạm thời                                                 |
| ----------------------------------- | ------------------------------------------------------------------- |
| Event id có bắt buộc không?         | Có. `eventId` là bắt buộc để Analytics chống xử lý trùng.           |
| Có cần correlationId không?         | Có. `correlationId` dùng để trace luồng từ Core sang Analytics.     |
| Có cho phép gửi trùng event không?  | Có thể xảy ra do retry. Analytics phải idempotent theo `eventId`.   |
| Event version đặt ở đâu?            | Dùng field `eventVersion`, ví dụ `1.0.0`.                           |
| Timestamp dùng chuẩn nào?           | ISO-8601 UTC, ví dụ `2026-05-19T10:00:00Z`.                         |
| `source` có giá trị gì?             | `core-business`.                                                    |
| `severity` dùng giá trị nào?        | `low`, `medium`, `high`, `critical`.                                |
| `decision` dùng giá trị nào?        | `allow`, `deny`, `review`.                                          |
| `resolutionNote` có bắt buộc không? | Không. Có thể null hoặc bỏ trống nếu chưa có ghi chú xử lý.         |
| Retry khi lỗi                       | Ghi rõ ở Lab 03.                                                    |
| Dead-letter queue                   | Ghi rõ ở Lab 03.                                                    |
| Ordering có bắt buộc không?         | Chưa bắt buộc ở Lab 02. Lab 03 sẽ làm rõ nếu cần theo `occurredAt`. |
| Consumer group                      | Ghi rõ ở Lab 03.                                                    |


## 6. Issue chuyển sang Lab 03

1. ...
2. ...
3. ...
