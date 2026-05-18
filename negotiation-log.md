# Biên bản đàm phán hợp đồng API/Event

- Cặp đàm phán:
  - Pair 06: IoT Ingestion → Analytics
  - Pair 07: Camera Stream → Analytics
  - Pair 08: Core Business → Analytics
  - Pair 09: Access Gate → Analytics
- Product: B
- Provider:
  - B1 — IoT Ingestion
  - B2 — Camera Stream
  - B6 — Core Business
  - B3 — Access Gate
- Consumer: B5 — Analytics
- Ngày: 2026-05-18

---

## Issue #1 — Thống nhất tên event/topic

- Raised by: Consumer
- Endpoint/Event:
  - `telemetry.ingested`
  - `device.status.changed`
  - `camera.motion.detected`
  - `camera.status.changed`
  - `alert.created`
  - `alert.resolved`
  - `access.log.created`
  - `access.denied`
- Concern: Nếu Provider publish sai tên event/topic, Analytics sẽ không nhận được dữ liệu.
- Proposal: Thống nhất tên event trước khi triển khai.
- Resolution: Accepted
- Rationale: Tên event thống nhất giúp các service tích hợp đúng.
- Impact: Provider publish đúng event name đã chốt, Consumer subscribe đúng topic.

---

## Issue #2 — Thiếu eventId gây duplicate metric

- Raised by: Consumer
- Endpoint/Event: All events
- Concern: Event có thể bị retry hoặc publish lại, khiến Analytics tính trùng metric.
- Proposal: Mọi event bắt buộc có `eventId`.
- Resolution: Accepted
- Rationale: `eventId` giúp Analytics nhận diện event trùng.
- Impact: Provider sinh `eventId`; Analytics dùng `eventId` để deduplicate.

---

## Issue #3 — Thiếu correlationId gây khó trace

- Raised by: Consumer
- Endpoint/Event: All events
- Concern: Khi báo cáo sai, nhóm khó truy vết event gốc qua nhiều service.
- Proposal: Mọi event nên có `correlationId`.
- Resolution: Accepted
- Rationale: `correlationId` giúp trace luồng xử lý giữa các service.
- Impact: Provider gửi `correlationId`; Analytics log theo `correlationId`.

---

## Issue #4 — Timestamp và timezone không thống nhất

- Raised by: Consumer
- Endpoint/Event: All events
- Concern: Nếu timestamp khác timezone hoặc format, Analytics aggregate sai theo giờ/ngày.
- Proposal: Dùng `timestamp` theo ISO 8601 UTC, ví dụ `2026-05-18T12:00:00Z`.
- Resolution: Accepted
- Rationale: Timestamp thống nhất giúp aggregate đúng theo giờ/ngày.
- Impact: Provider gửi timestamp chuẩn; Analytics aggregate theo timestamp của event.

---

## Issue #5 — IoT telemetry cần thống nhất unit

- Raised by: Consumer
- Endpoint/Event: `telemetry.ingested`
- Concern: Nếu unit không thống nhất, metric nhiệt độ/độ ẩm sẽ sai.
- Proposal:
  - `metricType`: `temperature`, `humidity`, `light`, `air_quality`
  - `unit`: `celsius`, `percent`, `lux`, `ppm`
- Resolution: Accepted
- Rationale: Unit thống nhất giúp Analytics tính toán đúng.
- Impact: IoT normalize unit trước khi gửi; Analytics validate unit trước khi tính metric.

---

## Issue #6 — Camera event không gửi binary image

- Raised by: Consumer
- Endpoint/Event:
  - `camera.motion.detected`
  - `camera.status.changed`
- Concern: Gửi ảnh trực tiếp trong event làm payload lớn, dễ timeout hoặc nghẽn broker.
- Proposal: Camera event chỉ gửi metadata; nếu cần ảnh thì gửi `imageRef`.
- Resolution: Accepted
- Rationale: Analytics chỉ cần metadata để thống kê.
- Impact: Camera Stream không gửi binary image trong event.

---

## Issue #7 — Core Business cần thống nhất severity và status

- Raised by: Consumer
- Endpoint/Event:
  - `alert.created`
  - `alert.resolved`
- Concern: Nếu severity/status không thống nhất, Analytics thống kê cảnh báo sai.
- Proposal:
  - `severity`: `low`, `medium`, `high`, `critical`
  - `status`: `open`, `resolved`, `ignored`
- Resolution: Accepted
- Rationale: Enum thống nhất giúp thống kê cảnh báo đúng.
- Impact: Core Business gửi severity/status đúng enum đã chốt.

---

## Issue #8 — Access Gate cần thống nhất direction và result

- Raised by: Consumer
- Endpoint/Event:
  - `access.log.created`
  - `access.denied`
- Concern: Nếu `direction` hoặc `result` không thống nhất, Analytics thống kê sai lượt vào/ra và lượt bị từ chối.
- Proposal:
  - `direction`: `entry` hoặc `exit`
  - `result`: `allowed` hoặc `denied`
- Resolution: Accepted
- Rationale: Enum thống nhất giúp aggregate đúng.
- Impact: Access Gate publish direction/result theo enum đã chốt.

---

## Issue #9 — Retry và dead-letter queue

- Raised by: Consumer
- Endpoint/Event: All events
- Concern: Event sai schema hoặc lỗi xử lý nếu retry vô hạn sẽ gây lặp và nghẽn hệ thống.
- Proposal:
  - Retry tối đa 3 lần.
  - Sau 3 lần lỗi, đưa event vào DLQ nếu hệ thống có hỗ trợ.
- Resolution: Accepted
- Rationale: DLQ giúp tách event lỗi để debug.
- Impact: Provider/Broker ghi nhận retry/DLQ policy.

---

## Issue #10 — Event ordering

- Raised by: Consumer
- Endpoint/Event: All events
- Concern: Event có thể đến không đúng thứ tự do Queue async.
- Proposal: Analytics aggregate theo `timestamp`, không dựa vào thứ tự nhận event.
- Resolution: Accepted
- Rationale: Giúp metric ổn định hơn khi event bị delay.
- Impact: Provider phải gửi timestamp chính xác.

---

# Chốt hợp đồng

Provider sign-off:

- B1 — IoT Ingestion:
- B2 — Camera Stream:
- B6 — Core Business:
- B3 — Access Gate:

Consumer sign-off:

- B5 — Analytics:

Witness:

Date: