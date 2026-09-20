# Phần 7 — Tổng hợp rủi ro & checklist vận hành

> Gom toàn bộ phát hiện từ Phần 1 → 6, xếp theo mức độ ảnh hưởng.
> Ngày phân tích: 2026-09-19

---

## 1. Rủi ro theo mức độ

### 🔴 Cao — mất dữ liệu / sai nghiệp vụ thầm lặng

| # | Vấn đề | Ở đâu | Hậu quả |
|---|---|---|---|
| 1 | **`providerId = null` lúc tạo rule vẫn trả 201** — nếu không có provider active theo `channelType`, binding lưu `providerId` rỗng | `DistributionRuleService.resolveProvider()` | Rule tạo OK, activate OK, trigger OK — nhưng **100% bản ghi `FAILED` + `terminal`**, không bao giờ gửi được, không bao giờ retry |
| 2 | **Phát voucher nằm ngoài transaction** — `dispatch()` không `@Transactional` | `RuleDispatchService.dispatch()` | pp-coupon đã cấp mã, `saveAllDistributions()` hỏng → khách có voucher trong ví mà **không nhận được tin nào**. Không có reconcile |
| 3 | **Outbox không nguyên tử nếu Mongo standalone** — URI hiện tại không phải replica set, `@Transactional` vô hiệu | `DistributionRuleService` + `application.properties:14` | Rule đã tạo nhưng mất event `DISTRIBUTION_RULE_CREATED` → downstream lệch vĩnh viễn |
| 4 | **Action lạ = gửi tin thường, im lặng** — chỉ `SEND_VOUCHER` được xử lý riêng | `RuleDispatchService.issueVoucherIfNeeded()` | Nếu seed thêm `ADD_POINTS` vào catalog mà quên sửa dispatch → rule chạy, gửi tin, **không cộng điểm**, không có lỗi nào |
| 5 | **Event thiếu `$.id` → không dedup** — `eventId` fallback sinh UUID mới mỗi lần | `RuleDispatchService.dispatch()` | Replay event = **phát mã mới + gửi tin lại** |

### 🟠 Trung bình — sai kết quả, phát hiện được

| # | Vấn đề | Ở đâu | Hậu quả |
|---|---|---|---|
| 6 | **Race trùng tên trả 500 thay vì 409** — `existsByName` rồi `save` không nguyên tử, không catch `DuplicateKeyException` | `DistributionRuleService.create()` | Request thua cuộc nhận 500. `update()` có catch tương đương, `create()` thì không |
| 7 | **Bug: placeholder trong `title` PUSH không resolve** — nhánh `if (templateType.equals("title"))` gán lại đúng giá trị cũ | `ProcessDistributionBatchService.augmentWithCatalog()` | Tiêu đề push hiện nguyên `{{key}}`. **Sửa 1 dòng** |
| 8 | **Placeholder không resolve được thì giữ nguyên, không log** | `MessageGenerationService.generateMessage()` | Khách nhận tin chứa `{{voucher_code}}` thô. Hay gặp khi gắn `{{voucher_code}}` vào rule `SEND_NOTIFICATION` |
| 9 | **Recipient không tìm được → `"unknown"`, vẫn gửi** | `ProcessDistributionBatchService.extractRecipient()` | Gọi provider với địa chỉ `"unknown"`, lỗi chỉ lộ ở phía nhà cung cấp |
| 10 | **Mọi lỗi HTTP pp-coupon quy về "Empty response"** — `onStatus` và `onErrorResume` đều `Mono.empty()` | `CouponPublishAdapter.publish()` | Không phân biệt pool hết mã (đừng retry) với lỗi mạng (nên retry) |
| 11 | **`campaignId` không validate lúc create** | `validateActionConfig()` | Sai campaign → phát hiện muộn ở runtime dưới dạng toàn bộ FAILED |
| 12 | **`eventTriggerCount` cộng trước khi lọc idempotency** | `RuleDispatchService.dispatch()` | Metric "Số lần kích hoạt" đếm cả event trùng lặp không sinh distribution nào |

### 🟡 Thấp — cần biết khi vận hành

| # | Vấn đề | Hậu quả |
|---|---|---|
| 13 | **`EventNotFoundException("")` khi mọi eventCode blank** — `@NotEmpty` chỉ chặn list rỗng | Trả 404 với identifier rỗng, đúng ra là 400 |
| 14 | **Provider mặc định lấy `active.get(0)`** — không có tiêu chí sắp xếp | Hai rule tạo ở hai thời điểm có thể bind về hai provider khác nhau |
| 15 | **Snapshot catalog lệch theo thời gian** — `providerName`, `templateCode`, `payloadPath` denormalize vào rule | Catalog đổi thì rule cũ giữ giá trị cũ cho tới khi update lại. Không có backfill |
| 16 | **`@Indexed(unique=true)` trên `name` xung đột với partial index mongock** | Vô hại hiện tại (auto-index-creation tắt mặc định), nhưng nếu bật sẽ chặn tạo lại rule trùng tên với rule đã soft-delete |
| 17 | **Consumer luôn `ack` kể cả khi bỏ qua** — không có DLQ | Event thiếu `type`/`customerId` mất luôn, chỉ còn `log.warn` |
| 18 | **Gọi pp-coupon đồng bộ trong consumer**, timeout 15s | N rule `SEND_VOUCHER` cùng khớp 1 event = N lần gọi tuần tự, tối đa N×15s consumer lag |
| 19 | **`action.config` của `SEND_NOTIFICATION` lưu mà không ai đọc** | Admin gửi gì cũng nhận 201, runtime bỏ qua |
| 20 | **Không cache rule** — query Mongo mỗi event | Ưu điểm: activate hiệu lực tức thì. Nhược: hot-path 1 round-trip/event |

---

## 2. Checklist "rule không chạy" — theo thứ tự kiểm

```
□ 1. status == RUNNING chưa?                    GET /{id} → status
□ 2. eventCode có consumer không?               đối chiếu Phần 2 §2 (chỉ 9 code)
       ⚠ ORDER_PAID KHÔNG có consumer riêng
□ 3. payloadPath có tiền tố $.payload. chưa?    log DEBUG ConditionValueEvaluator
□ 4. event có payload.customer_id không?        log.warn "missing payload.customer_id"
□ 5. condition khớp dữ liệu thật chưa?          log DEBUG "no match"
□ 6. có bị idempotency chặn không?              log INFO "Idempotency hit"
□ 7. channelBindings[].providerId có giá trị?   → nếu null: FAILED terminal, xem #1
□ 8. batch job có chạy không?                   log "Writing N distribution items"
```

### Lệnh grep phân tầng

```bash
cd p2_promotion-distribution/logs

# Tầng 1 — rule có được chọn không?
grep "DISPATCHING-VOUCHER-EVENT" *.log

# Tầng 2 — bản ghi có được tạo không?
grep "Saved .* distribution(s)" *.log

# Tầng 3 — batch có gửi không?
grep "Writing .* distribution items" *.log

# Các nút thắt hay gặp
grep -E "Idempotency hit|không gửi được|No applicable rules|Missing providerId" *.log
```

### Query Mongo chẩn đoán nhanh

```js
// Rule có đang RUNNING và bắt đúng event không?
db.distribution_rules.find(
  { "trigger.eventCodes": "ORDER_CREATED", deleted: { $ne: true } },
  { name: 1, status: 1, "channelBindings.channelType": 1, "channelBindings.providerId": 1 }
)

// Binding nào thiếu provider → sẽ FAILED terminal
db.distribution_rules.find({ "channelBindings.providerId": null, deleted: { $ne: true } }, { name: 1 })

// Bản ghi chết vĩnh viễn
db.distributions.aggregate([
  { $match: { terminal: true } },
  { $group: { _id: "$errorMessage", n: { $sum: 1 } } },
  { $sort: { n: -1 } }
])

// Bản ghi kẹt PENDING → batch không chạy
db.distributions.countDocuments({ status: "PENDING" })
```

---

## 3. Đề xuất sửa, theo thứ tự ưu tiên

| Ưu tiên | Sửa gì | Công sức |
|---|---|---|
| 1 | **Chặn tạo rule khi không resolve được provider** — hoặc trả 400, hoặc cảnh báo rõ trên response. Hiện tại 201 nhưng rule chết | nhỏ |
| 2 | **Bọc `saveAllDistributions` + phát voucher trong transaction**, hoặc thêm job reconcile mã đã cấp mà không có distribution | vừa |
| 3 | **Xác nhận Mongo có replica set** ở staging/prod; nếu không, `@Transactional` toàn module đang vô hiệu | điều tra |
| 4 | **Sửa dòng `templateType.equals("title")`** → `context.templateTitle()` | 1 dòng |
| 5 | **Catch `DuplicateKeyException` trong `create()`** → ném `DistributionRuleNameDuplicateException` | nhỏ |
| 6 | **Thêm case `SEND_NOTIFICATION` tường minh** trong dispatch (switch thay vì phủ định) để action lạ không im lặng trôi qua | nhỏ |
| 7 | **Log WARN khi placeholder không resolve được** thay vì im lặng giữ nguyên | nhỏ |
| 8 | **Phân loại lỗi trong `CouponPublishAdapter`** — tách lỗi mạng (retry được) khỏi lỗi nghiệp vụ | vừa |
| 9 | **`ORDER_PAID` và các event thiếu consumer**: hoặc bổ sung mapping, hoặc ẩn khỏi CMS để admin không chọn nhầm | vừa |

---

## 4. Điểm làm tốt (đừng phá khi refactor)

- **Aggregate root v3** — hot-path dispatch chỉ 1 query Mongo, không `$lookup`.
- **Ràng buộc "rule mới luôn PAUSED"** nằm trong domain model, đúng vị trí DDD.
- **Idempotency 2 tầng** — key ổn định ở cả distribution và pp-coupon.
- **1 mã / rule chứ không phải 1 mã / kênh** — key phát mã cố tình bỏ `channelType`.
- **`terminal` flag** — chặn bản ghi chắc chắn hỏng khỏi vòng retry vô ích.
- **Validation điều kiện ủy quyền cho platform validator** — một nguồn sự thật
  duy nhất về value-shape cho mọi module điều kiện PP.
- **Outbox pattern** thay vì publish Kafka trực tiếp trong service.
- **`TokenAcquisitionException` finalize tại chỗ** — tránh fail cả chunk rồi quét
  lại vô hạn.
