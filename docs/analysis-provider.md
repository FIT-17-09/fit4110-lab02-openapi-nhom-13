# Phân tích yêu cầu — vai Provider

- Cặp đàm phán:
  - Pair 06: IoT Ingestion → Analytics
  - Pair 07: Camera Stream → Analytics
  - Pair 08: Core Business → Analytics
  - Pair 09: Access Gate → Analytics
- Product: B
- Provider service:
  - B1 — IoT Ingestion
  - B2 — Camera Stream
  - B6 — Core Business
  - B3 — Access Gate
- Consumer service: B5 — Analytics
- Người viết: Nhóm B5 tổng hợp yêu cầu phía Provider cần cung cấp
- Ngày: 2026-05-18

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `TelemetryEvent` | Event dữ liệu cảm biến từ IoT Ingestion gửi sang Analytics | `eventId`, `eventType`, `timestamp`, `correlationId`, `deviceId`, `area`, `metricType`, `value`, `unit` | `roomId`, `floorId`, `quality`, `metadata` |
| `DeviceStatusEvent` | Event trạng thái thiết bị IoT | `eventId`, `eventType`, `timestamp`, `correlationId`, `deviceId`, `status` | `area`, `reason`, `lastSeenAt` |
| `CameraEvent` | Event camera phát hiện chuyển động, trạng thái camera hoặc kết quả xử lý frame | `eventId`, `eventType`, `timestamp`, `correlationId`, `cameraId`, `area`, `eventName` | `confidence`, `imageRef`, `processingTimeMs`, `status` |
| `AlertEvent` | Event cảnh báo/quyết định nghiệp vụ từ Core Business | `eventId`, `eventType`, `timestamp`, `correlationId`, `alertId`, `alertType`, `severity`, `status` | `area`, `reason`, `resolvedAt`, `durationSeconds` |
| `AccessLogEvent` | Event log ra/vào hoặc truy cập bị từ chối từ Access Gate | `eventId`, `eventType`, `timestamp`, `correlationId`, `accessLogId`, `gateId`, `direction`, `result` | `area`, `userId`, `cardHash`, `reason` |

---

## 2. Action/API dự kiến

Các cặp liên quan đến Analytics là Queue async. Provider publish event để Analytics subscribe.

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| N/A | `telemetry.ingested` | Gửi dữ liệu cảm biến cho Analytics | Khi có dữ liệu cảm biến mới |
| N/A | `device.status.changed` | Gửi trạng thái thiết bị IoT | Khi device online/offline hoặc lỗi |
| N/A | `camera.motion.detected` | Gửi event phát hiện chuyển động | Khi camera phát hiện chuyển động |
| N/A | `camera.status.changed` | Gửi trạng thái camera | Khi camera online/offline |
| N/A | `alert.created` | Gửi cảnh báo mới | Khi Core Business tạo alert |
| N/A | `alert.resolved` | Gửi trạng thái cảnh báo đã xử lý | Khi alert được resolved |
| N/A | `access.log.created` | Gửi log ra/vào | Khi có lượt vào/ra |
| N/A | `access.denied` | Gửi truy cập bị từ chối | Khi access bị denied |

---

## 3. Error case

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Event sai schema hoặc thiếu required field | `Problem` hoặc DLQ record |
| 401 | Consumer thiếu credential để đọc topic | `Problem` hoặc broker auth error |
| 403 | Consumer không có quyền subscribe topic | `Problem` hoặc broker permission error |
| 404 | Topic/event name không tồn tại | `Problem` hoặc broker topic not found |
| 409 | Event trùng `eventId` do retry | Consumer bỏ qua duplicate |
| 422 | Payload đúng JSON nhưng sai nghiệp vụ, ví dụ enum/unit không hợp lệ | `Problem` hoặc DLQ record |
| 500 | Provider/Broker lỗi nội bộ | Retry theo policy |
| 503 | Broker hoặc downstream tạm thời không sẵn sàng | Retry sau |

---

## 4. Giả định bổ sung

- Giả định 1: Provider publish event theo đúng tên đã thống nhất.
- Giả định 2: Mỗi event có `eventId` duy nhất để Analytics xử lý idempotency.
- Giả định 3: Mỗi event có `correlationId` để trace giữa các service.
- Giả định 4: `timestamp` dùng ISO 8601 UTC.
- Giả định 5: Provider không gửi binary image trực tiếp trong event.
- Giả định 6: Provider không gửi thông tin thẻ thô nếu không cần thiết.
- Giả định 7: Provider giữ nguyên field bắt buộc sau khi đã chốt contract.

---

## 5. Câu hỏi cho Consumer

1. Analytics cần aggregate telemetry theo `deviceId`, `area` hay cả hai?
2. Analytics có cần nhận batch telemetry không?
3. Analytics có bắt buộc cần `confidence` trong camera event không?
4. Analytics có cần `imageRef` trong camera event không?
5. Analytics muốn `reason` là text tự do hay reason code?
6. Analytics có cần `durationSeconds` khi alert resolved không?
7. Analytics muốn `direction` dùng `entry/exit` hay `in/out`?
8. Analytics có cần `cardHash` không?
9. Analytics xử lý duplicate event trong bao lâu?
10. Analytics có yêu cầu DLQ riêng không?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Tên event không thống nhất | Analytics không nhận được event | Chốt event name trong `negotiation-log.md` |
| Payload thiếu field bắt buộc | Analytics không aggregate được | Chốt required fields |
| Khác kiểu dữ liệu | Consumer parse lỗi | Chốt type và format |
| Không có `eventId` | Metric bị tính trùng | Bắt buộc event id |
| Không có `correlationId` | Khó trace lỗi | Bắt buộc correlation id |
| Timestamp khác timezone | Sai thống kê theo giờ/ngày | Dùng ISO 8601 UTC |
| Payload lớn | Broker chậm hoặc timeout | Không gửi binary, chỉ gửi reference |
| Retry không kiểm soát | Duplicate event | Consumer deduplicate theo `eventId` |
| Enum không thống nhất | Dashboard hiển thị sai metric | Chốt enum trong contract |