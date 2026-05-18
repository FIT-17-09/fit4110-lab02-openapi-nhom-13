# Phân tích yêu cầu — vai Consumer

- Cặp đàm phán:
  - Pair 06: IoT Ingestion → Analytics
  - Pair 07: Camera Stream → Analytics
  - Pair 08: Core Business → Analytics
  - Pair 09: Access Gate → Analytics
- Product: B
- Consumer service: B5 — Analytics
- Provider service:
  - B1 — IoT Ingestion
  - B2 — Camera Stream
  - B6 — Core Business
  - B3 — Access Gate
- Người viết: Nhóm B5
- Ngày: 2026-05-18

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `TelemetryEvent` | Nhận dữ liệu cảm biến từ IoT để tính nhiệt độ trung bình, độ ẩm, ánh sáng, chất lượng không khí theo khu vực/thời gian | `eventId`, `eventType`, `timestamp`, `correlationId`, `source`, `deviceId`, `area`, `metricType`, `value`, `unit` | `roomId`, `floorId`, `quality`, `metadata` |
| `CameraEvent` | Nhận dữ liệu camera để thống kê phát hiện chuyển động, trạng thái camera và event bất thường | `eventId`, `eventType`, `timestamp`, `correlationId`, `source`, `cameraId`, `area`, `eventName` | `confidence`, `imageRef`, `processingTimeMs`, `status` |
| `AlertEvent` | Nhận dữ liệu cảnh báo/quyết định từ Core Business để thống kê số cảnh báo theo ngày, mức độ và trạng thái xử lý | `eventId`, `eventType`, `timestamp`, `correlationId`, `source`, `alertId`, `alertType`, `severity`, `status` | `area`, `reason`, `resolvedAt`, `durationSeconds` |
| `AccessLogEvent` | Nhận dữ liệu ra/vào từ Access Gate để thống kê lượt vào, lượt ra, lượt bị từ chối và giờ cao điểm | `eventId`, `eventType`, `timestamp`, `correlationId`, `source`, `accessLogId`, `gateId`, `direction`, `result` | `area`, `userId`, `cardHash`, `reason` |

---

## 2. API Consumer cần gọi

Các cặp liên quan đến Analytics là Queue async. Analytics không gọi REST API trực tiếp trong luồng chính, mà nhận event do Provider gửi.

| Method | Path | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| N/A | `telemetry.ingested.v1` | Khi IoT Ingestion gửi dữ liệu cảm biến mới | Analytics nhận event có `deviceId`, `area`, `metricType`, `value`, `unit`, `timestamp` |
| N/A | `device.status.changed.v1` | Khi trạng thái thiết bị IoT thay đổi | Analytics nhận event có `deviceId`, `status`, `timestamp` |
| N/A | `camera.motion.detected` | Khi Camera Stream phát hiện chuyển động | Analytics nhận event có `cameraId`, `area`, `eventName`, `timestamp` |
| N/A | `camera.status.changed` | Khi camera online/offline | Analytics nhận event có `cameraId`, `status`, `timestamp` |
| N/A | `alert.created` | Khi Core Business tạo cảnh báo | Analytics nhận event có `alertId`, `alertType`, `severity`, `timestamp` |
| N/A | `alert.resolved` | Khi Core Business xử lý xong cảnh báo | Analytics nhận event có `alertId`, `status`, `resolvedAt` |
| N/A | `access.log.created` | Khi Access Gate tạo log ra/vào | Analytics nhận event có `accessLogId`, `gateId`, `direction`, `result`, `timestamp` |
| N/A | `access.denied` | Khi Access Gate từ chối truy cập | Analytics nhận event có `accessLogId`, `gateId`, `reason`, `timestamp` |

Ghi chú: Vì đây là Queue async nên `Method` và `Path` không dùng HTTP method/path thật. Tên trong cột `Path` được hiểu là tên event/topic dự kiến.

---

## 3. Error case Consumer cần xử lý

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| 400 | Event/payload sai schema hoặc thiếu field bắt buộc | Ghi log lỗi, reject event, không đưa vào metric |
| 401 | Consumer thiếu credential để đọc queue/topic | Kiểm tra credential/token, báo lỗi cấu hình |
| 403 | Consumer có credential nhưng không có quyền đọc queue/topic | Báo lỗi phân quyền, yêu cầu cấp quyền topic |
| 404 | Topic/queue hoặc event name không tồn tại | Kiểm tra lại tên event/topic đã thống nhất |
| 409 | Event bị trùng `eventId` do retry hoặc publish lặp | Bỏ qua event trùng, không aggregate lại |
| 422 | Payload đúng JSON nhưng sai nghiệp vụ, ví dụ `unit`, `direction`, `severity` không hợp lệ | Ghi log chi tiết, không đưa event vào metric |
| 500 | Lỗi nội bộ khi Analytics xử lý event | Retry có giới hạn, nếu vẫn lỗi thì đưa vào DLQ nếu có |
| 503 | Broker hoặc dependency tạm thời không sẵn sàng | Tạm dừng consume và retry sau |

---

## 4. Giả định bổ sung

- Giả định 1: Tất cả event gửi tới Analytics đều có `eventId` để chống xử lý trùng.
- Giả định 2: Tất cả event đều có `correlationId` để trace luồng xử lý.
- Giả định 3: Thời gian event dùng `timestamp` theo chuẩn ISO 8601 UTC, ví dụ `2026-05-18T12:00:00Z`.
- Giả định 4: Analytics aggregate theo `timestamp` của event, không dựa vào thứ tự nhận event.
- Giả định 5: Event có thể bị gửi lại do retry, Analytics phải idempotent theo `eventId`.
- Giả định 6: Dữ liệu cảm biến từ IoT dùng unit thống nhất.
- Giả định 7: Camera event không gửi ảnh nhị phân trực tiếp, nếu cần thì gửi `imageRef`.
- Giả định 8: Access Gate không gửi thông tin thẻ thô, nếu cần thì dùng `cardHash`.

---

## 5. Câu hỏi cho Provider

1. IoT Ingestion có thống nhất các `metricType` là `temperature`, `humidity`, `light`, `air_quality` không?
2. IoT Ingestion có thống nhất unit là `°C`, `%`, `lux`, `ppm`, `μg/m³`, `Pa` không?
3. Camera Stream có gửi `confidence` trong event phát hiện chuyển động không?
4. Camera Stream có gửi ảnh thật trong event không, hay chỉ gửi `imageRef`?
5. Core Business có thống nhất `severity` là `low`, `medium`, `high`, `critical` không?
6. Core Business có gửi `resolvedAt` khi cảnh báo được xử lý không?
7. Access Gate dùng `direction` là `entry/exit` hay `in/out`?
8. Access Gate dùng `result` là `allowed/denied` không?
9. Provider có hỗ trợ retry hoặc DLQ khi event lỗi không?
10. Provider có đảm bảo mỗi event có `eventId` duy nhất không?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider đổi kiểu dữ liệu | Consumer parse lỗi | Chốt type/format trong contract |
| Event thiếu `eventId` | Analytics không chống trùng được | Bắt buộc `eventId` trong mọi event |
| Event thiếu `correlationId` | Khó trace lỗi giữa các service | Bắt buộc `correlationId` |
| Timestamp khác format/timezone | Aggregate sai theo giờ/ngày | Dùng ISO 8601 UTC |
| Unit IoT không thống nhất | Tính nhiệt độ/độ ẩm sai | Chốt enum unit |
| Enum Access Gate không thống nhất | Thống kê vào/ra sai | Chốt `direction` và `result` |
| Camera gửi ảnh trực tiếp | Payload lớn, dễ lỗi queue | Chỉ gửi `imageRef` |
| Event đến không đúng thứ tự | Metric realtime có thể sai tạm thời | Aggregate theo `timestamp` |
| Event bị retry nhiều lần | Metric bị cộng trùng | Deduplicate theo `eventId` |
| Topic/event name không thống nhất | Analytics không nhận được dữ liệu | Chốt tên event/topic trong `negotiation-log.md` |