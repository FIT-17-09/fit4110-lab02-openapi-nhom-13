# Biên bản đàm phán hợp đồng API/Event — B5 Analytics

- Product: B
- Service chính: B5 — Analytics Service
- Nhóm viết: B5
- Phiên bản: v1.0
- Ngày cập nhật: 2026-05-25

---

## 1. Tổng quan các cặp tích hợp của B5

| Pair | Provider | Consumer | Cơ chế | Mục đích |
|---|---|---|---|---|
| Pair 06 | B1 — IoT Ingestion | B5 — Analytics | Queue async | Feed telemetry cho aggregate |
| Pair 07 | B2 — Camera Stream | B5 — Analytics | Queue async | Feed event camera cho aggregate |
| Pair B4-B5 | B4 — AI Vision | B5 — Analytics | Queue async/Event JSON | Feed kết quả AI analysis cho dashboard |
| Pair 08 | B6 — Core Business | B5 — Analytics | Queue async | Feed alert/decision/policy event cho KPI |
| Pair 09 | B3 — Access Gate | B5 — Analytics | Queue async | Feed log ra/vào cho thống kê |
| Pair B7-B5 | B7 — Notification | B5 — Analytics | Queue async | Feed notification event cho KPI/report |

Ghi chú:

- Các cặp Queue async trong Lab 02 chưa cần AsyncAPI đầy đủ.
- Mục tiêu Lab 02 là ghi nhận event contract sơ bộ: event/topic, producer, consumer, payload tối thiểu, `eventId`, `timestamp`, `correlationId`, retry/DLQ.

---

# 2. Pair 06 — B1 IoT Ingestion → B5 Analytics

## Issue #1 — Thống nhất tên event IoT

- Raised by: Provider B1 và Consumer B5
- Event/Topic:
  - `telemetry.ingested`
- Concern: Nếu B1 và B5 dùng tên event khác nhau, Analytics sẽ không subscribe đúng dữ liệu telemetry.
- Proposal:
  - B1 publish event telemetry bằng tên `telemetry.ingested`.
  - Nếu cần version event, dùng suffix `.v1` theo convention của B1 trong bước phát triển tiếp theo.
- Resolution: Accepted
- Rationale: Tên event thống nhất giúp B5 nhận đúng luồng dữ liệu cảm biến.
- Impact:
  - B1 publish đúng event name.
  - B5 subscribe đúng topic/event name.

---

## Issue #2 — Thống nhất unit sensor

- Raised by: Provider B1
- Event/Topic:
  - `telemetry.ingested`
- Concern: Sensor có thể gửi nhiều đơn vị khác nhau như °C/°F, Pa/bar. Nếu không normalize, B5 aggregate sai metric.
- Proposal:
  - B1 normalize unit trước khi publish.
  - Unit hợp lệ:
    - `°C` cho nhiệt độ
    - `Pa` cho áp suất
    - `lux` cho ánh sáng
    - `ppm` hoặc `μg/m³` cho chất lượng không khí
    - `%` cho độ ẩm
  - Unit không hợp lệ sẽ đưa vào DLQ.
- Resolution: Accepted
- Rationale: Normalize tại B1 giúp B5 không phải tự convert dữ liệu từ nhiều loại sensor.
- Impact:
  - B1 chịu trách nhiệm convert/normalize unit.
  - B5 validate unit trước khi aggregate.

---

## Issue #3 — Idempotency cho telemetry event

- Raised by: Provider B1
- Event/Topic:
  - All IoT events
- Concern: Queue async có thể retry, khiến B5 nhận trùng event và aggregate sai.
- Proposal:
  - Mỗi event phải có `eventId`.
  - `eventId` là string UUID v4.
  - B5 dùng `eventId` làm idempotency key.
  - Dedup window tối thiểu 24 giờ.
- Resolution: Accepted
- Rationale: `eventId` giúp B5 bỏ qua event trùng khi broker retry.
- Impact:
  - B1 sinh `eventId` duy nhất cho mỗi event.
  - B5 lưu event đã xử lý để deduplicate.

---

## Issue #4 — Correlation ID cho tracing

- Raised by: Provider B1
- Event/Topic:
  - All IoT events
- Concern: Khi metric sai hoặc có lỗi xử lý, cần trace event từ device → IoT → Analytics.
- Proposal:
  - Mỗi event có `correlationId`.
  - Nếu upstream chưa có, B1 tự sinh UUID.
- Resolution: Accepted
- Rationale: `correlationId` giúp trace end-to-end giữa các service.
- Impact:
  - B1 truyền `correlationId`.
  - B5 log theo `correlationId`.

---

## Issue #5 — Batch telemetry

- Raised by: Consumer B5
- Event/Topic:
  - `telemetry.ingested`
- Concern: B5 muốn nhận từng event riêng lẻ để aggregate realtime, nhưng B1 có thể muốn gửi batch.
- Proposal:
  - V1 trong Lab 02 chỉ nhận event đơn.
  - Mỗi event có `value` và `timestamp`.
  - `batchId` là optional field để chuẩn bị cho V2.
- Resolution: Modified
- Rationale: Lab 02 giữ event đơn để đơn giản, nhưng vẫn mở đường cho batch ở Lab sau.
- Impact:
  - B1 publish từng telemetry event.
  - B5 chưa cần xử lý `readings[]` batch trong Lab 02.

---

## Issue #6 — Retry, DLQ và out-of-order

- Raised by: Provider B1 và Consumer B5
- Event/Topic:
  - All IoT events
- Concern:
  - Payload sai schema có thể làm hỏng pipeline.
  - Retry vô hạn có thể gây loop.
  - Event có thể đến không đúng thứ tự.
- Proposal:
  - Retry tối đa 3 lần.
  - Backoff: 1s, 2s, 4s.
  - DLQ: `campus.iot.dlq`.
  - DLQ retention: 7 ngày.
  - DLQ record có `errorType`, `errorMessage`, `failedAt`.
  - Timestamp dùng ISO 8601 UTC, ví dụ `2026-05-18T12:00:00Z`.
  - B5 aggregate theo `timestamp`, không dựa vào thứ tự nhận event.
- Resolution: Accepted
- Rationale: DLQ giúp tách event lỗi; timestamp UTC giúp thống kê đúng theo thời gian.
- Impact:
  - B1 cấu hình retry/DLQ.
  - B5 xử lý late-arriving event theo cửa sổ thời gian.

---

# 3. Pair 07 — B2 Camera Stream → B5 Analytics

## Issue #7 — Dữ liệu camera cho Analytics

- Raised by: Consumer B5
- Event/Topic:
  - `camera.motion.detected`
  - `camera.status.changed`
- Concern: B5 cần event camera để thống kê số lần phát hiện chuyển động, trạng thái camera và các event bất thường.
- Proposal:
  - B2 publish metadata camera event cho B5.
  - Không gửi binary image trực tiếp trong event.
  - Nếu cần ảnh, chỉ gửi `imageRef` hoặc `image_url`.
- Resolution: Need confirm from B2
- Rationale: Metadata đủ cho Analytics aggregate, tránh payload lớn.
- Impact:
  - B2 cần xác nhận lại topic/event name và payload.
  - B5 aggregate theo `cameraId`, `area`, `timestamp`.

---

## Issue #8 — Payload camera tối thiểu

- Raised by: Consumer B5
- Event/Topic:
  - `camera.motion.detected`
- Concern: Nếu thiếu camera context, B5 khó aggregate theo khu vực/camera.
- Proposal:
  - Required fields:
    - `eventId`
    - `correlationId`
    - `timestamp`
    - `eventType`
    - `cameraId`
    - `area`
    - `eventName`
  - Optional fields:
    - `imageRef`
    - `confidence`
    - `status`
    - `metadata`
- Resolution: Need confirm from B2
- Rationale: Các field này đủ để B5 thống kê số motion event và camera status.
- Impact:
  - B2 cần xác nhận field bắt buộc.
  - B5 xử lý duplicate bằng `eventId`.

---

# 4. Pair B4-B5 — B4 AI Vision → B5 Analytics

## Issue #9 — Event kết quả phân tích AI

- Raised by: Consumer B5
- Event/Topic:
  - `vision.analysis.completed`
- Concern: B5 cần dữ liệu phân tích từng frame ảnh để vẽ biểu đồ mật độ, phát hiện bất thường và KPI theo camera/khu vực.
- Proposal:
  - Khi B4 phân tích xong 1 frame, B4 push event JSON sang B5.
  - Payload bám theo chuẩn event chung của hệ thống.
- Resolution: Accepted
- Rationale: B5 có dữ liệu realtime để aggregate AI metric.
- Impact:
  - B4 publish event `vision.analysis.completed`.
  - B5 consume event để tính metric AI.

Payload thống nhất:

```json
{
  "eventId": "uuid-v4",
  "correlationId": "uuid-v4",
  "timestamp": "2026-05-20T10:30:02Z",
  "eventType": "vision.analysis.completed",
  "data": {
    "cameraId": "cam-gate-01",
    "detected": true,
    "object": "person",
    "confidence": 0.95,
    "riskLevel": "low"
  }
}
```

---

## Issue #10 — Tracing và idempotency với AI event

- Raised by: B4 và B5
- Event/Topic:
  - `vision.analysis.completed`
- Concern: AI event có thể bị retry hoặc cần trace từ lúc camera chụp ảnh đến khi B5 xử lý.
- Proposal:
  - Bắt buộc truyền `correlationId` xuyên suốt từ B2 → B4 → B5.
  - B4 tự sinh `eventId` UUID cho mỗi event gửi sang B5.
  - B5 deduplicate bằng `eventId`.
- Resolution: Accepted
- Rationale: Tránh lặp dữ liệu khi retry và hỗ trợ debug cross-service.
- Impact:
  - B4 đảm bảo `eventId` và `correlationId`.
  - B5 lưu event đã xử lý để deduplicate.

---

# 5. Pair 08 — B6 Core Business → B5 Analytics

## Issue #11 — SLA và topic Core Business event

- Raised by: Provider B6 và Consumer B5
- Topic/Queue:
  - `core-business-events`
- Concern: Core Business gửi alert/decision/policy event cho Analytics; cần thống nhất SLA, retention và delivery guarantee.
- Proposal:
  - Communication type: Queue async RabbitMQ/Kafka.
  - Throughput: ≥ 500 events/sec.
  - Latency: P95 ≤ 5s, P99 ≤ 10s.
  - Delivery: at-least-once.
  - Availability: 99.5%.
  - Data retention: 30 ngày.
- Resolution: Accepted
- Rationale: Analytics không yêu cầu realtime tuyệt đối nhưng cần dữ liệu đủ để aggregate KPI.
- Impact:
  - B6 publish vào `core-business-events`.
  - B5 consume bằng consumer group `analytics-service`.

---

## Issue #12 — Event format và validation từ Core Business

- Raised by: Provider B6
- Topic/Queue:
  - `core-business-events`
- Concern: Payload cần thống nhất để B5 aggregate alert/decision chính xác.
- Proposal:
  - Message format: JSON UTF-8.
  - Max payload size: 512 KB.
  - Timestamp format: ISO 8601.
  - Schema version được track trong event payload.
  - No sensitive PII in events.
- Resolution: Accepted
- Rationale: Chuẩn hóa format giúp B5 parse và validate ổn định.
- Impact:
  - B6 gửi rich metadata cần thiết.
  - B5 validate schema trước khi aggregate.

---

## Issue #13 — Delivery, retry, DLQ và monitoring cho Core Business event

- Raised by: B6 và B5
- Topic/Queue:
  - `core-business-events`
- Concern:
  - Queue async có thể duplicate.
  - Event lỗi cần DLQ.
  - Consumer lag cần monitor.
- Proposal:
  - At-least-once delivery.
  - B5 deduplicate bằng `eventId`.
  - Dedup window: 30 ngày.
  - Publisher retries: 3 lần, backoff 100ms, 500ms, 1s.
  - Consumer retries: 3 lần, backoff 1s, 2s, 4s.
  - Failed messages → DLQ.
  - DLQ retention: 30 ngày.
  - B5 commit offset sau khi aggregation hoàn tất.
- Resolution: Accepted
- Rationale: Đảm bảo không mất dữ liệu và tránh tính trùng metric.
- Impact:
  - B6 publish durable event.
  - B5 monitor lag, DLQ và commit offset thủ công.

---

# 6. Pair 09 — B3 Access Gate → B5 Analytics

## Issue #14 — Log ra/vào cho Analytics

- Raised by: Consumer B5
- Event/Topic:
  - `access.log.created`
  - `access.denied`
- Related REST endpoints from B3:
  - `/access/logs/recent`
  - `/access/logs/{logId}`
- Concern: B5 cần dữ liệu ra/vào để thống kê lượt vào, lượt ra, lượt bị từ chối và giờ cao điểm.
- Proposal:
  - B3 cung cấp access log theo contract Access Gate đã thống nhất.
  - Nếu có event pipeline, B3 publish access event dạng Queue async cho B5.
  - Nếu chưa có event pipeline, B5 dùng REST/schema log của B3 làm chuẩn dữ liệu.
  - B5 aggregate theo `gateId`, `area`, `direction`, `result`, `timestamp`.
- Resolution: Accepted
- Rationale: Analytics cần log ra/vào để phục vụ dashboard và báo cáo.
- Impact:
  - B3 cung cấp log truy cập theo schema thống nhất.
  - B5 aggregate access metric từ log/event của Access Gate.

---

## Issue #15 — Payload Access Gate tối thiểu

- Raised by: Consumer B5
- Event/Topic:
  - `access.log.created`
  - `access.denied`
- Concern: B5 cần đủ field để aggregate theo cổng, khu vực, chiều di chuyển và kết quả truy cập.
- Proposal:
  - Required fields:
    - `eventId`
    - `correlationId`
    - `timestamp`
    - `eventType`
    - `accessLogId`
    - `gateId`
    - `direction`
    - `result`
  - Enum:
    - `direction`: `IN`, `OUT`
    - `result`: `ALLOWED`, `DENIED`
  - Optional fields:
    - `area`
    - `userId`
    - `cardHash`
    - `cardStatus`
    - `operatorNote`
    - `reason`
  - B3-specific rules:
    - `operatorNote` có thể nullable.
    - `cardStatus`: `ACTIVE`, `BLOCKED`, `EXPIRED`.
    - Access log retention tối thiểu 30 ngày.
    - REST timeout tối đa 3 giây.
    - `/health` không yêu cầu Bearer token.
- Resolution: Accepted
- Rationale: Các field này đủ để B5 thống kê access metric và khớp với contract của B3.
- Impact:
  - B3 cung cấp/publish log theo field đã thống nhất.
  - B5 mapping `IN` thành lượt vào, `OUT` thành lượt ra.
  - B5 deduplicate bằng `eventId`.

---

# 7. Pair B7-B5 — B7 Notification → B5 Analytics

## Issue #16 — Topic notification event

- Raised by: B7 và B5
- Topic/Queue:
  - `notification-events`
- Concern: B5 cần notification event để tracking notification metrics, KPI aggregation, dashboard analytics, monitoring/reporting và audit/tracing.
- Proposal:
  - B7 publish notification event qua Queue async.
  - B5 consume topic `notification-events`.
- Resolution: Accepted
- Rationale: Queue async phù hợp để tracking notification metrics mà không ảnh hưởng luồng gửi thông báo.
- Impact:
  - B7 publish event.
  - B5 aggregate theo notification status/channel/type.

---

## Issue #17 — Timestamp UTC cho notification event

- Raised by: Consumer B5
- Topic/Queue:
  - `notification-events`
- Concern: Nếu timestamp khác timezone, dashboard aggregate sai theo ngày/giờ.
- Proposal:
  - Tất cả timestamp dùng ISO 8601 UTC.
  - Ví dụ: `2026-05-21T08:30:00Z`.
- Resolution: Accepted
- Rationale: UTC giúp đồng bộ nhiều service.
- Impact:
  - B7 gửi timestamp UTC.
  - B5 aggregate theo timestamp UTC.

---

## Issue #18 — Metadata-only payload

- Raised by: Provider B7
- Topic/Queue:
  - `notification-events`
- Concern: Payload notification quá lớn nếu chứa full email/SMS/push content.
- Proposal:
  - Không gửi full email body, SMS content hoặc push notification detail.
  - Chỉ gửi metadata:
    - `eventId`
    - `notificationId`
    - `eventType`
    - `channel`
    - `status`
    - `timestamp`
    - `correlationId`
    - `retryCount`
- Resolution: Accepted
- Rationale: B5 chỉ cần metadata để aggregate metrics, không cần nội dung thông báo.
- Impact:
  - B7 giảm payload size.
  - B5 không lưu nội dung nhạy cảm.

---

## Issue #19 — Event type của Notification

- Raised by: Consumer B5
- Topic/Queue:
  - `notification-events`
- Concern: B5 cần phân biệt nhiều loại notification để filter dashboard.
- Proposal:
  - Supported event types:
    - `NOTIFICATION_SENT`
    - `NOTIFICATION_FAILED`
    - `BROADCAST_TRIGGERED`
    - `SYSTEM_ALERT_SENT`
    - `USER_NOTICE_CREATED`
- Resolution: Accepted
- Rationale: Event classification giúp Analytics aggregate theo category.
- Impact:
  - B7 gửi `eventType`.
  - B5 group metric theo `eventType`.

---

## Issue #20 — Deduplication, DLQ và offset commit cho Notification

- Raised by: B7 và B5
- Topic/Queue:
  - `notification-events`
- Concern:
  - At-least-once delivery có thể sinh duplicate.
  - Payload lỗi cần route riêng.
  - B7 không muốn tốc độ publish bị ảnh hưởng bởi Analytics.
- Proposal:
  - Deduplication key: `eventId`.
  - Deduplication window: 24 giờ.
  - Retry attempts: 3.
  - DLQ: `notification-events-dlq`.
  - B5 commit offset sau khi xử lý bất đồng bộ hoàn tất.
  - B7 gửi thêm `retryCount` để B5 tracking reliability.
- Resolution: Accepted
- Rationale: Đảm bảo B5 không đếm trùng và không làm ảnh hưởng tốc độ publish của Notification.
- Impact:
  - B7 publish retry-safe event.
  - B5 xử lý async safely và commit offset sau khi aggregate xong.

---

# 8. Quy chuẩn chung cho toàn bộ B5 Analytics

## Issue #21 — Chuẩn field chung cho tất cả event gửi vào B5

- Raised by: B5
- Event/Topic:
  - All events
- Concern: Mỗi nhóm đặt tên field khác nhau sẽ làm B5 parse lỗi hoặc aggregate sai.
- Proposal:
  - Mọi event gửi vào B5 nên có tối thiểu:
    - `eventId`
    - `correlationId`
    - `timestamp`
    - `eventType`
  - Payload nghiệp vụ đặt trong `data` nếu Provider hỗ trợ.
  - Timestamp dùng ISO 8601 UTC.
  - Event phải là JSON UTF-8.
- Resolution: Accepted / Need provider confirm
- Rationale: Chuẩn field chung giúp B5 viết consumer thống nhất.
- Impact:
  - B5 xử lý được nhiều event từ nhiều service.
  - Provider giảm rủi ro mismatch contract.

---

## Issue #22 — Deduplication và late-arriving event

- Raised by: B5
- Event/Topic:
  - All events
- Concern: Queue async có thể duplicate hoặc event đến muộn.
- Proposal:
  - B5 deduplicate bằng `eventId`.
  - B5 aggregate theo `timestamp`, không dựa vào thời điểm nhận event.
  - Với event đến muộn, B5 xử lý theo windowed aggregation.
- Resolution: Accepted
- Rationale: Tránh sai metric trong môi trường async.
- Impact:
  - B5 cần lưu event processed.
  - B5 cần xử lý late-arriving event.

---

## Issue #23 — Error handling chung

- Raised by: B5
- Event/Topic:
  - All events
- Concern: Payload lỗi nếu không tách riêng sẽ làm hỏng pipeline aggregate.
- Proposal:
  - Payload sai JSON/schema/required field → retry tối đa 3 lần.
  - Sau retry → DLQ tương ứng của từng Provider.
  - B5 log lỗi gồm:
    - `eventId`
    - `correlationId`
    - `eventType`
    - `errorType`
    - `errorMessage`
    - `failedAt`
- Resolution: Accepted / Need provider confirm
- Rationale: DLQ giúp debug mà không block luồng chính.
- Impact:
  - B5 không aggregate event lỗi.
  - Provider có thể review DLQ để sửa payload.

---

# 9. Sign-off

## Provider sign-off

- B1 — IoT Ingestion: Accepted
- B2 — Camera Stream: Need confirm
- B3 — Access Gate: Accepted
- B4 — AI Vision: Accepted, chờ B5 xác nhận cuối
- B6 — Core Business: Accepted
- B7 — Notification: Accepted

## Consumer sign-off

- B5 — Analytics: Accepted

## Witness

- GV/TA:

## Date

- 2026-05-25

---