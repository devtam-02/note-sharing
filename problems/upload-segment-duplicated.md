# Phân tích: `CUSTOMER_ENTERED_SEGMENT` bị gửi 2–3 lần cho cùng một khách

> **Ngày**: 2026-10-04
> **Nhánh**: `tamntt25/staging-local` (HEAD `4031ae2`)
> **Triệu chứng**: khách hàng chỉ upload **một** file CSV **một** lần, nhưng distribution gửi event `CUSTOMER_ENTERED_SEGMENT` 2–3 lần cho cùng một customer.
> **Trạng thái**: phân tích từ code; **chưa có log hay dữ liệu thật** của case để xác nhận. Xem mục 5 để biết cách xác minh.

---

## 1. Kết luận nhanh

- Distribution **đã có chống trùng** theo id của event segment. Vì vậy việc gửi lặp lại cùng một event Kafka **không** gây trùng.
- Để khách nhận 2–3 lần, module segment phải đã tạo ra **2–3 `SegmentMembershipAddedEvent` khác `id`** cùng chứa khách đó. Điều này chỉ xảy ra khi **cùng một lô bị `processBulkAdd` xử lý nhiều lần**, hoặc khi khách nằm ở nhiều lô.
- Nghi vấn có khả năng cao nhất: **Kafka rebalance ở consumer `SegmentCommandConsumer`**, khiến các lô của file bị xử lý lại. Mỗi lần xử lý lại tạo một event với `id` mới.

---

## 2. Event được tạo ra thế nào

```
segment ── ProcessSegmentMembershipBulkAddService#processBulkAdd  (1 lần / lô 500 khách)
   └─ publishSuccessEvents
        └─ EventPublisherAdapter#publishSegmentMembershipAddedEvent
             SegmentMembershipEvent { id = IdGenerator.generateId()  ← MỚI mỗi lần gọi,
                                      type = "SegmentMembershipAddedEvent", payload.customerIds[] }
        └─ OutboxEventPublisher → collection outbox_events (status PENDING)
        └─ OutboxProcessorScheduler (30s) → outboxProcessorJob → Kafka "promotion_segment_event"

distribution ── SegmentMembershipConsumer  (p2_promotion-distribution)
   └─ "SegmentMembershipAddedEvent" → code "CUSTOMER_ENTERED_SEGMENT"
   └─ mỗi customerId → RuleDispatchService#dispatch(eventId = "<id event segment>:<customerId>")
        └─ chống trùng: idempotencyKey = topic + eventId + ruleId + channelType
           (filterPendingBindings → findByIdempotencyKey → "Idempotency hit — skip")
```

- Trong module segment, **chỉ có một nơi** gọi `publishSegmentMembershipAddedEvent`: `ProcessSegmentMembershipBulkAddService#publishSuccessEvents`.
- `id` của event được sinh ngẫu nhiên mỗi lần, **không** gắn với `asyncId`/`batchId`. Xử lý lại cùng một lô sẽ ra event mới, nên distribution không nhận ra là trùng.

---

## 3. Các nghi vấn

### ① Kafka rebalance làm consumer segment xử lý lại lô — **khả năng cao nhất**

| Yếu tố | Giá trị / vị trí |
|---|---|
| Key của message lô | `segmentId` (`SegmentMembershipWriter` → `SegmentEventProducer`), nên mọi lô của một file vào **cùng một partition** và được **xử lý tuần tự** |
| Số record mỗi lần poll | `promix.messaging.kafka.consumer.max-poll=500`. Property `segment.kafka.consumer.max-poll-records=100` **không được code nào dùng**. Việc `max-poll` của promix có map sang `max.poll.records` hay không thì **chưa kiểm tra** trong thư viện |
| Thời gian tối đa giữa 2 lần poll | `promix.messaging.kafka.consumer.max-poll-interval=300000` (5 phút) |
| Ack | `MANUAL_IMMEDIATE`. `SegmentCommandConsumer#consumeCommand` gọi `acknowledge()` sau khi xử lý xong, lỗi thì ném exception |
| Thời gian một lô | Khoảng 9 giây hoặc hơn trước commit `b292beb` (`upsertAll` quét toàn bộ collection) |

**Cơ chế**:
1. Consumer poll tối đa 500 lô, rồi xử lý lần lượt trước khi poll tiếp.
2. Nếu tổng thời gian vượt 5 phút thì broker loại consumer khỏi group, và partition được giao cho consumer khác.
   - Với khoảng 9 giây mỗi lô, chỉ cần **khoảng 34 lô, tức là file khoảng 17 nghìn dòng**, là đã vượt.
3. Consumer mới đọc lại từ offset đã commit gần nhất. Commit của consumer cũ sau thời điểm bị loại đều thất bại. Trong lúc đó consumer cũ có thể vẫn chạy nốt các lô đã poll.
4. Mỗi lần rebalance là một lần xử lý lại, nên **2–3 lần là khớp** với triệu chứng.

**Khi một lô bị xử lý lại thì chuyện gì xảy ra**:

| Bước | Có chống trùng không | Kết quả |
|---|---|---|
| Cập nhật tiến độ Redis (`incrementDataProgressIdempotent`) | Có, key `processedBatchKey:batchId` | Không cộng lại |
| `upsertAll` membership | Có, unique index `(segmentId, customerId)` | Không tạo trùng |
| `createAuditHistory` | **Không** | Ghi thêm history `ADDED` |
| `updateSearchDocuments` (ES) | Không có hại | Ghi đè |
| `publishSuccessEvents` | **Không** | **Thêm một `SegmentMembershipAddedEvent` với `id` mới**, nên distribution gửi `CUSTOMER_ENTERED_SEGMENT` lần nữa |
| `batch_processing_logs` | Không | Thêm một bản ghi log cho cùng `batchId` |

### ② Khách có mặt ở nhiều lô của file

- Trong file có `sourceId` trùng nhưng `SegmentMembershipProcessor` không bắt được. Processor là **singleton**, giữ `processedSourceIds` dùng chung cho mọi job. Trong khi đó cấu hình cho phép chạy song song tối đa 5 job (`promix.batch.concurrency-limit=5`). Khi một job khác bắt đầu, `beforeStep` của nó **xoá sạch** Set của job đang chạy.
- Hoặc hai `sourceId` chỉ khác nhau về chữ hoa/chữ thường: processor so sánh có phân biệt hoa thường, nhưng nếu DB customer không phân biệt thì cả hai map về cùng một `customerId`. Điểm này **chưa kiểm tra** collation của DB customer.

### ③ Spring Batch chạy lại một chunk

Nếu commit chunk thất bại, chẳng hạn do lỗi ghi JobRepository trên MariaDB, chunk sẽ được chạy lại. Khi đó `SegmentMembershipWriter#write` thêm các dòng đó vào `buffer` một lần nữa, trong khi buffer nằm ngoài transaction. Khả năng thấp.

### Đã loại trừ

| Nghi vấn | Lý do loại |
|---|---|
| Outbox gửi cùng một event nhiều lần, ví dụ nhiều pod cùng chạy `outboxProcessorJob` (job này đọc rồi lưu, không có claim nguyên tử) | Event gửi lặp lại vẫn **cùng `id`**, nên distribution chặn bằng idempotencyKey. Đây vẫn là rủi ro gửi Kafka trùng, nhưng không phải nguyên nhân khiến khách nhận nhiều lần |
| Bộ xử lý outbox của thư viện `promix` (`OutboxScheduledProcessor`, bật mặc định, dùng chung collection `outbox_events`) | `KafkaOutboxEventPublisher` bắt buộc có field `destination`. Document của app lưu ở field `topic`, nên thư viện **không gửi được** event của app. Có thể gây **mất** event, không gây trùng (xem mục 6) |
| Kafka gửi lại message phía distribution | Cùng `eventId` nên bị chặn |

---

## 4. Ảnh hưởng của commit `b292beb` tới nghi vấn ①

`b292beb` sửa query `deletedAt` để dùng được partial index, nên `upsertAll` giảm từ khoảng 9 giây xuống mức dự đoán vài chục đến vài trăm ms. Thời gian mỗi lô giảm mạnh, nên **rủi ro rebalance giảm theo**, nhưng **không hết**. Ví dụ: 500 lô × 1 giây = 500 giây, vẫn lớn hơn 300 giây. Cần xác định case xảy ra trước hay sau khi `b292beb` được deploy.

---

## 5. Cách xác minh

Thay `<asyncId>`, `<segmentId>`, `<customerId>` bằng giá trị thật của case.

```js
// (a) Có lô nào bị xử lý nhiều lần không? Có kết quả → xác nhận nghi vấn ①
db.batch_processing_logs.aggregate([
  { $match: { asyncId: "<asyncId>" } },
  { $group: { _id: "$batchId", n: { $sum: 1 }, starts: { $push: "$startedAt" } } },
  { $match: { n: { $gt: 1 } } }
])

// (b) Có bao nhiêu SegmentMembershipAddedEvent chứa khách này, id có khác nhau không
//     (payload lưu dạng chuỗi JSON nên tìm bằng regex được)
db.outbox_events.find(
  { eventType: "SegmentMembershipAddedEvent", aggregateId: "<segmentId>", payload: /<customerId>/ },
  { _id: 1, createdAt: 1, status: 1 }
)

// (c) Số bản ghi history ADDED (3 bản ghi → đã xử lý 3 lần)
db.segment_membership_history.find({ segmentId: "<segmentId>", customerId: "<customerId>" })

// (d) Khách có bị loại vì trùng trong file không (nghi vấn ②)
db.segment_import_errors.find({ asyncId: "<asyncId>", errorType: "DUPLICATE_SOURCE_ID" })
```

Trong log của segment, tìm theo `asyncId` hoặc khoảng thời gian import:

| Log | Ý nghĩa |
|---|---|
| `BATCH_SKIP: Batch already processed (idempotency)` | Cùng một `batchId` đi vào consumer lần thứ hai, tức là bị xử lý lại |
| `Revoking`, `partitions revoked`, `rebalance`, `CommitFailedException`, `max.poll.interval` | Xác nhận rebalance (nghi vấn ①) |
| `COMMAND_FAILED` | Consumer ném exception, Kafka sẽ gửi lại message |

Bên distribution: các `distribution` của khách có `idempotencyKey` chứa **nhiều `eventId` khác nhau** thì khớp với kết luận ở mục 1.

---

## 6. Rủi ro phụ phát hiện thêm (chưa phải nguyên nhân của case)

1. **Hai bộ xử lý outbox cùng chạy trên `outbox_events`**: `OutboxProcessorScheduler` của app chạy mỗi 30 giây, và `OutboxScheduledProcessor` của thư viện chạy mỗi 3 giây, do `promix.outbox.mongo.enabled=true` mà `promix.outbox.scheduler.enabled` mặc định bật.
   - Bộ của thư viện nhận được document `PENDING` của app, nhưng gửi thất bại vì thiếu `destination`, rồi đánh dấu `FAILED` hoặc `DEAD_LETTER`.
   - Bộ của app chỉ đọc `PENDING`, nên event có thể **bị mất** nếu thư viện nhận trước.
   - Nên tắt một trong hai bộ, ví dụ đặt `promix.outbox.scheduler.enabled=false`, hoặc tách collection.
2. **`outboxProcessorJob` không nhận event một cách nguyên tử**. Nó đọc toàn bộ `PENDING` vào `ListItemReader` rồi mới lưu, không có `@Version` hay `findAndModify`. Khi chạy nhiều pod, cùng một event có thể được gửi Kafka nhiều lần, với cùng `id`. Distribution chặn được, nhưng các consumer khác của topic thì chưa chắc.
3. **`segment_membership_history` bị ghi `ADDED` trùng** mỗi lần lô bị xử lý lại, do unique index gồm cả `created_at`.

---

## 7. Hướng sửa đề xuất

| # | Thay đổi | Tác dụng | Độ lớn |
|---|---|---|---|
| 1 | Tạo `id` của `SegmentMembershipAddedEvent` **cố định theo lô**, ví dụ hash(`asyncId` + `batchId`), thay cho `IdGenerator.generateId()` | Lô bị xử lý lại cho ra cùng `id`, distribution tự chặn bằng idempotencyKey sẵn có. Chặn trùng với mọi nguyên nhân xử lý lại | Nhỏ |
| 2 | Đầu `processBulkAdd`: nếu `batchId` đã `COMPLETED` (theo `batch_processing_logs` hoặc Redis) thì **bỏ qua cả lô** | Tránh luôn history trùng, ES ghi thừa, và lỗi `ALREADY_IN_SEGMENT` báo nhầm (xem mục 8) | Vừa |
| 3 | Giảm `max.poll.records` riêng cho listener `SegmentCommandConsumer`, ví dụ 10–20, hoặc tăng `max.poll.interval.ms` | Tránh rebalance ngay từ gốc | Nhỏ, chỉ đổi config |
| 4 | Chuyển `SegmentMembershipProcessor` sang `@StepScope` | Sửa nghi vấn ②: hết dùng chung Set giữa các job chạy song song | Nhỏ |
| 5 | Tắt một trong hai bộ xử lý outbox | Sửa rủi ro mất event ở mục 6.1 | Nhỏ, chỉ đổi config |

Thứ tự gợi ý: **1 + 3** trước (nhỏ, chặn được triệu chứng), sau đó **2** và **4**.

---

## 8. Tổng quan tác dụng của commit `4031ae2`

> `4031ae2` (tamntt25, 2026-09-29): *feat: report customers already in segment as ALREADY_IN_SEGMENT on CSV import*.
> 3 file: `ProcessSegmentMembershipBulkAddService.java` (+198), `ImportErrorType.java` (+5), `ProcessSegmentMembershipBulkAddExistingMemberTest.java` (+266, 6 test).

### 8.1 Mục đích

Trước commit, luồng import CSV **không kiểm tra** khách đã thuộc segment chưa. Khách cũ được xử lý như khách mới:
- tính là thành công,
- ghi thêm history `ADDED`,
- bắn lại `SegmentMembershipAddedEvent`,
- và bị gửi sang `bulk-upsert` của customer service. Bước này **ghi đè tên** và phát `CustomerUpdatedEvent` cho mỗi khách.

### 8.2 Thay đổi trong `processBulkAdd`

```
1. findCustomersAlreadyInSegment   ── validateSourceIds (API /validate, chỉ đọc) → sourceId → customerId
                                      findBySegmentIdAndCustomerIds → khách đã có trong segment
2. upsertCustomers(chỉ khách còn lại)  ── khách đã có trong segment KHÔNG gửi sang bulk-upsert
3. mergeAlreadyInSegment           ── đưa họ vào failedCustomers, errorCode ALREADY_IN_SEGMENT
4. excludeExistingMembers          ── lưới an toàn sau upsert: dùng khi validateSourceIds lỗi
                                      (nó trả về "không tìm thấy ai" thay vì ném exception)
5. Các bước sau giữ nguyên          ── tiến độ, membership, history, ES, event, lưu lỗi
```

### 8.3 Tác dụng

| Khía cạnh | Trước | Sau |
|---|---|---|
| Kết quả của khách đã có trong segment | Thành công | **Lỗi** `ALREADY_IN_SEGMENT` ("Khách hàng đã tồn tại trong segment"), lưu ở `segment_import_errors` |
| Tiến độ Redis | Tính thành công | Tính thất bại |
| Gửi sang customer service `bulk-upsert` | Có, nên bị ghi đè tên và phát `CustomerUpdatedEvent` | **Không** |
| Membership | Upsert, chỉ đổi `updatedAt` | Không động tới |
| History `ADDED` | Ghi thêm | Không ghi |
| ES | Ghi đè | Không động tới |
| `SegmentMembershipAddedEvent`, dẫn tới `CUSTOMER_ENTERED_SEGMENT` ở distribution | **Có**, nên khách cũ có thể nhận lại voucher hoặc thông báo | **Không** |
| Event thất bại qua outbox (`SegmentMembershipAddFailedEvent`) | Không | Có thêm các khách này |

### 8.4 Chi phí

- Mỗi lô thêm **1 lời gọi HTTP** `/validate` sang customer service. `validateSourceIds` không có circuit breaker.
- Mỗi lô thêm **2 query** `findBySegmentIdAndCustomerIds`, chạy trên partial index. Hai query này chỉ rẻ khi đã có commit `b292beb`.
- Bù lại, customer service không phải upsert những khách đã có trong segment.

### 8.5 Liên quan tới case gửi trùng

- **Giảm triệu chứng một cách tình cờ.** Khi một lô bị xử lý lại (nghi vấn ①), khách vừa được thêm ở lần đầu sẽ bị nhận diện là `ALREADY_IN_SEGMENT`, nên không vào event "đã thêm". Vì vậy distribution không gửi `CUSTOMER_ENTERED_SEGMENT` lần nữa.
- **Nhưng sinh lỗi báo nhầm.** Ở lần xử lý lại, các khách đó bị ghi lỗi `ALREADY_IN_SEGMENT` trong `segment_import_errors`, dù thực chất là khách mới của chính lần import này. Trên CMS sẽ thấy số dòng lỗi tăng sai. Tiến độ Redis thì không bị ảnh hưởng, vì lần xử lý lại bị chặn theo `batchId`.
- Vì vậy `4031ae2` **không phải** bản sửa cho case gửi trùng. Vẫn cần hướng sửa 1 và 2 ở mục 7. Riêng hướng 2 (bỏ qua lô đã `COMPLETED`) còn loại bỏ luôn lỗi báo nhầm này.

### 8.6 Hạn chế đã biết

1. Nếu cả `validateSourceIds` lẫn query Mongo đều lỗi, lô chạy như trước khi có commit: khách cũ lại được tính thành công. Có log `ALREADY_IN_SEGMENT_CHECK_FAILED` / `EXISTING_MEMBER_CHECK_FAILED`.
2. Hai file cùng segment import song song: bước kiểm tra trước khi upsert không nguyên tử. Dữ liệu vẫn an toàn nhờ unique index, nhưng một khách có thể được tính thành công ở cả hai file.
3. Vấn đề **ghi đè tên khách bằng tên rỗng hoặc hỏng** khi upload chưa được xử lý. Commit chỉ tránh được với khách đã có trong segment. Xem mục 7.10 của `docs/upload-csv-flow.md`.
