# Phần 8 — Sơ đồ đầy đủ luồng phân phối

> Toàn cảnh end-to-end trên một tài liệu: **3 cửa vào → chọn rule → rẽ nhánh action
> → ghi `distributions` → batch gửi tin → kết quả**, kèm mọi nhánh lỗi, retry, DLQ
> và ranh giới thread.
>
> Module `p2_promotion-distribution` · Cập nhật: 2026-09-21
> Bổ sung so với Phần 3–6: **cửa vào thứ 2 và thứ 3**, DLQ, cấu hình batch thật,
> backoff của provider, ranh giới thread.

---

## 1. Toàn cảnh một trang

```mermaid
flowchart TB
    subgraph IN["CỬA VÀO (3 đường)"]
        K1["Kafka · 6 topic<br/>6 @KafkaListener"]
        K2["REST POST /redemption-events<br/>RedemptionEventController"]
        K3["Outbox relay<br/>(rule lifecycle — KHÔNG tạo distribution)"]
    end

    subgraph DISPATCH["TẦNG 1 — DISPATCH (thread Kafka consumer)"]
        RD["RuleDispatchService.dispatch()"]
        PR["ProcessRedemptionEventService.process()<br/>@Transactional @Retryable(3)"]
        EV["DistributionRuleEvaluationService<br/>findApplicableRules()"]
        MG1[("Mongo<br/>distribution_rules<br/>status=RUNNING")]
        ACT{"action.code<br/>== SEND_VOUCHER?"}
        CP["CouponPublishAdapter<br/>→ pp-coupon"]
    end

    subgraph STORE["KHO"]
        MG2[("Mongo<br/>distributions")]
    end

    subgraph BATCH["TẦNG 2 — GỬI TIN (thread batch, mỗi 5s)"]
        SCH["DistributionJobScheduler<br/>@Scheduled(fixedRate=5000)"]
        RE["Reader — PENDING + FAILED(retry<3)"]
        PRO["Processor — enrich rule/provider/template"]
        BS["ProcessDistributionBatchService<br/>recipient + {{placeholder}}"]
        WR["Writer"]
        PS["ProviderService.sendMessage()"]
        EXT["Nhà cung cấp<br/>SMS / PUSH"]
        MARIA[("MariaDB<br/>batch metadata")]
    end

    subgraph OUT["ĐẦU RA"]
        KOUT["Kafka promotion_distribution_events<br/>DISTRIBUTION_SUCCESS"]
        DLQ["Kafka &lt;topic&gt;_dlqmo"]
    end

    K1 --> RD
    K2 --> PR
    RD --> EV
    PR --> EV
    EV --> MG1
    EV --> ACT
    ACT -- "có" --> CP
    CP --> MG2
    ACT -- "không" --> MG2
    RD -. "exception" .-> DLQ

    SCH --> RE
    MG2 --> RE
    RE --> PRO --> BS --> WR --> PS --> EXT
    PS --> WR
    WR --> MG2
    WR --> KOUT
    SCH -.-> MARIA

    style ACT fill:#ffe8cc,stroke:#c83,stroke-width:2px
    style MG2 fill:#efe,stroke:#393,stroke-width:2px
    style DLQ fill:#fee,stroke:#c33
    style K3 fill:#eee,stroke:#999
```

> **Cửa vào thứ 3 không tạo distribution**: outbox `promotion_distribution_rule_event`
> chỉ phát sự kiện vòng đời rule (CREATED/UPDATED/ACTIVATED/PAUSED/DELETED) cho hệ
> downstream. Grep toàn workspace: **không có consumer nào**. Vẽ ở đây để khỏi nhầm.

---

## 2. Cửa vào chi tiết

### 2.1 — Sáu consumer Kafka

```mermaid
flowchart LR
    T1["promotion_order_event"] --> C1["OrderEventConsumer"]
    T2["promotion_voucher_event"] --> C2["VoucherEventConsumer"]
    T3["promotion_segment_event"] --> C3["SegmentMembershipConsumer"]
    T4["promotion-cashback-events"] --> C4["CashbackEventConsumer"]
    T5["promotion_redemption_confirmed"] --> C5["RedemptionConfirmedConsumer"]
    T6["promotion_custom_event"] --> C6["CustomEventConsumer"]

    C1 --> M1["OrderCreatedEvent<br/>→ ORDER_CREATED"]
    C2 --> M2["VoucherRedeemedEvent → VOUCHER_REDEEMED<br/>VoucherPublishedEvent → SUCCESSFULLY_PUBLISHED<br/>VoucherRevokedEvent → VOUCHER_REVOKED"]
    C3 --> M3["SegmentMembershipAddedEvent → CUSTOMER_ENTERED_SEGMENT<br/>SegmentMembershipRemovedEvent → CUSTOMER_LEFT_SEGMENT"]
    C4 --> M4["CashbackSuccessEvent<br/>→ CASHBACK_SUCCEEDED"]
    C5 --> M5["hardcode<br/>→ REWARD_REDEEMED"]
    C6 --> M6["$.type dùng trực tiếp<br/>→ BẤT KỲ code nào"]

    M1 & M2 & M3 & M4 & M5 & M6 --> D["RuleDispatchService.dispatch()"]

    style C6 fill:#e8f0ff,stroke:#369,stroke-width:2px
```

Mỗi consumer làm đúng 4 bước rồi `ack`:

```
1. map type → eventCode      → thiếu: log.warn + ack + return   ⚠ MẤT MESSAGE
2. lấy payload.customer_id   → thiếu: log.warn + ack + return   ⚠ MẤT MESSAGE
3. serialize lại rawJson
4. dispatch(DispatchCommand)
```

### 2.2 — Cửa REST (ít ai biết)

```mermaid
flowchart LR
    A["POST {context-path}/redemption-events<br/>RedemptionEventController:35"] --> B["MessageDto<br/>type = 'RedemptionEvent'"]
    B --> C["ProcessRedemptionEventService.process()<br/>@Transactional @Retryable(maxAttempts=3, delay=1s)"]
    C --> D{"isValid?<br/>promotionId + customerId"}
    D -- "không" --> E["log.error + InvalidRedemptionEventException"]
    D -- "có" --> F["findApplicableRules('RedemptionEvent', payload)"]
    F --> G["mỗi rule × mỗi binding → Distribution PENDING"]
    G --> H["saveAllDistributions()"]

    style A fill:#ffe8cc,stroke:#c83
```

Khác biệt so với đường Kafka — **cần biết để không hiểu nhầm khi debug**:

| | Đường Kafka (`RuleDispatchService`) | Đường REST (`ProcessRedemptionEventService`) |
|---|---|---|
| `@Transactional` | ❌ không | ✅ có, `rollbackFor = Exception.class` |
| Retry | qua Kafka error handler + DLQ | `@Retryable(3, backoff 1s)` tại chỗ |
| Lọc idempotency | ✅ `filterPendingBindings()` | ❌ **không có** |
| Đếm `eventTriggerCount` | ✅ | ❌ **không đếm** |
| Phát voucher `SEND_VOUCHER` | ✅ | ❌ **không phát** — dù rule là SEND_VOUCHER |
| `eventCode` dùng để match | từ bảng map của consumer | luôn là chuỗi `"RedemptionEvent"` |

> ⚠️ Đường REST **bỏ qua hoàn toàn nhánh action**. Rule `SEND_VOUCHER` khớp qua cửa
> này sẽ gửi tin mà **không có mã**, `{{voucher_code}}` giữ nguyên chuỗi thô.
> Javadoc của service còn tham chiếu `ProcessCashbackEventService` — class này
> **không còn tồn tại**.

---

## 3. Tầng 1 — Dispatch đầy đủ

```mermaid
flowchart TD
    A["DispatchCommand<br/>topic, eventCode, eventId, customerId, payload, rawJson"] --> B{"eventCode rỗng?"}
    B -- "có" --> Z0["return 0"]
    B -- "không" --> C{"eventId rỗng?"}
    C -- "có" --> C1["eventId = IdGenerator.generateId()<br/>⚠ MỚI mỗi lần → mất dedup"]
    C -- "không" --> D
    C1 --> D["findApplicableRules(eventCode, payload)"]

    D --> D1[("Mongo: trigger.eventCodes = code<br/>AND status = RUNNING<br/>AND deleted != true")]
    D1 --> D2["ConditionValueEvaluator.matches()<br/>JSONPath + AND logic, fail-safe"]
    D2 --> E{"applicable rỗng?"}
    E -- "có" --> Z1["log: No applicable rules<br/>return 0"]

    E -- "không" --> F["FOR EACH rule"]
    F --> G["incrementEventTriggerCount(ruleId, 1)<br/>$inc atomic"]
    G --> H["filterPendingBindings()<br/>key = topic:eventId:ruleId:channelType"]
    H --> H1{"còn binding nào?"}
    H1 -- "không" --> Z2["log: All deduped<br/>return 0"]

    H1 -- "có" --> I{"action.code == SEND_VOUCHER?"}
    I -- "KHÔNG<br/>(SEND_NOTIFICATION / null / khác)" --> J["voucher = null<br/>eventJson = rawJson"]
    I -- "CÓ" --> K["issueVoucherIfNeeded()"]

    K --> K1{"campaignId có?"}
    K1 -- "không" --> KF["PublishResult.failed<br/>Missing campaignId"]
    K1 -- "có" --> K2{"customerId có?"}
    K2 -- "không" --> KF2["PublishResult.failed<br/>Missing customerId"]
    K2 -- "có" --> K3["pp-coupon POST /vouchers/publish<br/>Idempotency-Key = eventId:ruleId:customerId<br/>ĐỒNG BỘ, timeout 15s"]
    K3 --> K4{"status PUBLISHED<br/>&& voucherCode?"}
    K4 -- "có" --> K5["ok(code)<br/>→ injectVoucherCode vào $.payload.voucher_code"]
    K4 -- "không" --> KF3["failed(reason)"]

    J --> L["FOR EACH binding → buildDistribution()"]
    K5 --> L
    KF --> L
    KF2 --> L
    KF3 --> L

    L --> M{"resolveBlockingError()"}
    M -- "voucher thất bại" --> N1["FAILED + terminal=true"]
    M -- "providerId rỗng" --> N2["FAILED + terminal=true"]
    M -- "null (OK)" --> N3["PENDING"]

    N1 & N2 & N3 --> O["saveAllDistributions()<br/>⚠ KHÔNG try/catch, KHÔNG @Transactional"]
    O -- "Mongo lỗi" --> P["exception → Kafka error handler<br/>retry → DLQ &lt;topic&gt;_dlqmo"]
    O -- "OK" --> Q[("distributions")]

    style I fill:#ffe8cc,stroke:#c83,stroke-width:2px
    style N1 fill:#fee,stroke:#c33
    style N2 fill:#fee,stroke:#c33
    style P fill:#fee,stroke:#c33
    style Q fill:#efe,stroke:#393
```

### Bốn nhánh xử lý lỗi của consumer

```mermaid
flowchart LR
    A["Message tới consumer"] --> B{"Kết cục"}
    B -- "1. Thành công" --> S["ack.acknowledge()<br/>AckMode MANUAL_IMMEDIATE"]
    B -- "2. Thiếu type/customerId" --> W["log.warn → ack → return<br/>❌ KHÔNG exception ⇒ KHÔNG vào DLQ<br/>⇒ MẤT VĨNH VIỄN"]
    B -- "3. Mongo/provider ném exception" --> E["CommittingErrorHandler<br/>retry theo backOff<br/>→ DLQ &lt;topic&gt;_dlqmo → commit offset"]
    B -- "4. Pod chết khi PROCESSING" --> P["❌ KHÔNG job nào reset<br/>Reader chỉ đọc PENDING/FAILED<br/>⇒ KẸT VĨNH VIỄN"]

    style W fill:#fee,stroke:#c33,stroke-width:2px
    style P fill:#fee,stroke:#c33,stroke-width:2px
    style E fill:#ffe,stroke:#ca3
```

---

## 4. Tầng 2 — Pipeline gửi tin đầy đủ

```mermaid
flowchart TD
    SCH["DistributionJobScheduler<br/>@Scheduled(fixedRate = 5000)<br/>@ConditionalOnProperty promix.batch.enabled=true"] --> JOB["scanDistributionsJob<br/>RunIdIncrementer"]
    JOB --> STEP["scanDistributionsStep<br/>chunk=100 · concurrency=5 · faultTolerant<br/>skipLimit=5 · retryLimit=0"]
    STEP -.-> MARIA[("MariaDB<br/>batch metadata<br/>datasource riêng")]

    STEP --> R["DistributionMongoItemReader<br/>status=PENDING<br/>OR (FAILED AND retry&lt;3 AND terminal!=true)<br/>sort: retry ASC, createdAt ASC"]

    R --> P1["DistributionItemProcessor.enrich()"]
    P1 --> E1{"ruleId có?"}
    E1 -- "không" --> FAIL["ProcessedDistribution FAILED"]
    E1 -- "có" --> E2{"rule findById?"}
    E2 -- "không" --> FAIL
    E2 -- "có" --> E3{"binding khớp channelType?"}
    E3 -- "không" --> FAIL
    E3 -- "có" --> E4{"providerId có?"}
    E4 -- "không" --> FAIL
    E4 -- "có" --> E5{"provider tồn tại & active?"}
    E5 -- "không" --> FAIL
    E5 -- "có" --> E6["nội dung = binding.content<br/>rỗng → template.body"]

    E6 --> BS["ProcessDistributionBatchService.processDistribution()"]
    BS --> BS1["extractRecipient()"]
    BS1 --> BS2["extractMessageParams() → resolve {{placeholder}}"]
    BS2 --> BS3["generateMessage(template, params)"]
    BS3 --> OK["status = PROCESSING"]

    OK --> W["DistributionItemWriter.write()"]
    FAIL --> W
    W --> W1{"status == PROCESSING?"}
    W1 -- "không" --> W2["createFailedResponse()"]
    W1 -- "có" --> PS["ProviderService.sendMessage()"]

    PS --> UP["updateDistributionStatuses()<br/>updateFirst _id, inc version"]
    W2 --> UP
    UP --> KP["publishSuccessEvents()<br/>chỉ SUCCESS → Kafka"]

    style FAIL fill:#fee,stroke:#c33
    style OK fill:#efe,stroke:#393
```

### 4.1 — Xác định người nhận

```mermaid
flowchart TD
    A["JSONPath theo channelType"] --> A1["SMS → $.payload.customerPhone<br/>EMAIL → $.payload.customerEmail<br/>PUSH → $.payload.deviceId<br/>khác → $.recipient"]
    A1 --> B{"có giá trị?"}
    B -- "không" --> C["fallback $.payload.customerPhone"]
    C --> D{"có?"}
    B -- "có" --> OK["dùng"]
    D -- "có" --> OK
    D -- "không" --> E["pp-customer getCustomerProfile(customerId)"]
    E --> F{"SMS → phone<br/>EMAIL → email<br/>khác → phone"}
    F -- "có" --> OK
    F -- "không" --> G["recipient = 'unknown'<br/>⚠ VẪN GỬI"]

    style G fill:#fee,stroke:#c33
```

### 4.2 — Resolve `{{placeholder}}`

```mermaid
flowchart TD
    K["{{key}} trong nội dung"] --> T0{"Tầng 0<br/>staticParams[] của rule?"}
    T0 -- "LITERAL" --> V0["literalValue"]
    T0 -- "SOURCE" --> V0b["pp-campaign theo sourceId"]
    T0 -- "không có" --> T1{"Tầng 1<br/>bindingParamsLogic?"}
    T1 -- "có" --> V1["JSONPath mapping (legacy)"]
    T1 -- "không" --> T2["Tầng 2 — catalog message_parameters"]

    T2 --> T2a{"resolvePaths[] trên event?"}
    T2a -- "có" --> V2a["giá trị từ payload<br/>← {{voucher_code}} vào đây"]
    T2a -- "không" --> T2b{"enrichField?"}
    T2b -- "có" --> V2b["pp-customer: name/phone/email"]
    T2b -- "không" --> T2c{"campaignField?"}
    T2c -- "có" --> V2c["pp-campaign"]
    T2c -- "không" --> T2d{"key == current_date?"}
    T2d -- "có" --> V2d["LocalDate.now()"]
    T2d -- "không" --> V2e["defaultValue"]
    V2e --> NULL{"vẫn null?"}
    NULL -- "có" --> KEEP["⚠ GIỮ NGUYÊN chuỗi {{key}}<br/>không log, không fail"]

    style V2a fill:#e8f0ff,stroke:#369
    style KEEP fill:#fee,stroke:#c33
```

### 4.3 — Gửi và phân loại kết quả

```mermaid
flowchart TD
    A["ProviderService.sendMessage()"] --> M{"mock-enabled?"}
    M -- "true" --> MK["sendMessageMock()<br/>success-rate 0.7, delay 1s"]
    M -- "false" --> B["protocolResolver.resolve()"]
    B --> B1{"protocol"}
    B1 -- "JSON_REST (hoặc null)" --> S1["JsonRestSendStrategy"]
    B1 -- "SMS_BANKING_SOAP" --> S2["SmsBankingSoapSendStrategy"]
    B1 -- "không xác định" --> PF["permanentFailure<br/>Unusable provider protocol"]

    S1 & S2 --> T{"cần token?"}
    T -- "có" --> T1["acquireTokenIfNeeded()"]
    T1 -- "TokenAcquisitionException" --> HE["handleProviderError()<br/>finalize tại chỗ, KHÔNG ném ra Writer"]
    T -- "không" --> EX["strategy.send()"]
    T1 --> EX

    EX --> R{"kết quả"}
    R -- "2xx" --> SU["SUCCESS<br/>→ Kafka promotion_distribution_events"]
    R -- "401 && retry-on-401" --> RT["invalidateToken → refreshToken → send lại"]
    R -- "lỗi tạm thời" --> TF["handleTemporaryFailure()"]
    R -- "lỗi vĩnh viễn" --> PF

    TF --> TF1{"retryCount+1 >= 5?"}
    TF1 -- "có" --> FF["FAILED<br/>Max retries exceeded"]
    TF1 -- "không" --> BO["PENDING + retryCount+1<br/>executedAt = now + backoff<br/>backoff = min(2^n × 30s, 30 phút)"]

    style SU fill:#efe,stroke:#393,stroke-width:2px
    style FF fill:#fee,stroke:#c33
    style PF fill:#fee,stroke:#c33
```

---

## 5. Vòng đời trạng thái `Distribution`

```mermaid
stateDiagram-v2
    [*] --> PENDING : dispatch OK
    [*] --> FAILED_TERMINAL : resolveBlockingError()<br/>voucher hỏng / thiếu providerId

    PENDING --> PROCESSING : processor enrich OK
    PENDING --> FAILED : enrich lỗi (rule/provider/template)

    PROCESSING --> SUCCESS : provider 2xx
    PROCESSING --> FAILED : max retry / lỗi vĩnh viễn
    PROCESSING --> PENDING : lỗi tạm thời<br/>retry+1, backoff 2^n×30s

    PROCESSING --> STUCK : ⚠ pod chết giữa chừng

    FAILED --> PROCESSING : reader đọc lại<br/>(retry<3 && terminal!=true)
    FAILED --> [*] : hết lượt retry

    SUCCESS --> [*] : + Kafka DISTRIBUTION_SUCCESS
    FAILED_TERMINAL --> [*] : KHÔNG BAO GIỜ retry
    STUCK --> [*] : ⚠ KHÔNG job nào cứu

    note right of STUCK
        Reader chỉ đọc PENDING và FAILED.
        Không có scheduler reset PROCESSING.
    end note
```

> ⚠️ **Hai ngưỡng retry lệch nhau**: reader lọc `retry < 3`
> (`distribution.batch.max-retries=3`) nhưng `ProviderService` chỉ đánh FAILED khi
> `retry >= 5` (`distribution.retry.max-attempts=5`). Bản ghi đạt `retry = 3` và 4
> bị **reader bỏ qua trong khi ProviderService vẫn coi là còn lượt** → nằm im ở
> `PENDING`/`FAILED` mà không ai xử lý.

---

## 6. Sequence đầy đủ (nhánh SEND_VOUCHER)

```mermaid
sequenceDiagram
    autonumber
    participant K as Kafka
    participant C as Consumer
    participant RD as RuleDispatchService
    participant EV as EvaluationService
    participant M as Mongo
    participant CP as pp-coupon
    participant SCH as Batch 5s
    participant PR as Processor
    participant BS as BatchService
    participant CU as pp-customer
    participant PS as ProviderService
    participant EXT as Provider SMS/PUSH

    K->>C: event (topic, type, payload)
    Note over C: map type → eventCode<br/>lấy payload.customer_id
    C->>RD: dispatch(DispatchCommand)

    RD->>EV: findApplicableRules(eventCode, payload)
    EV->>M: query trigger.eventCodes + RUNNING
    M-->>EV: candidates[]
    Note over EV: ConditionValueEvaluator<br/>JSONPath + AND
    EV-->>RD: applicable[]

    loop mỗi rule
        RD->>M: $inc eventTriggerCount
        RD->>M: findByIdempotencyKey × mỗi binding
        M-->>RD: binding còn lại

        Note over RD: action.code == SEND_VOUCHER
        RD->>CP: POST /vouchers/publish<br/>Idempotency-Key
        CP-->>RD: { status: PUBLISHED, voucherCode }
        Note over RD: inject $.payload.voucher_code
    end

    RD->>M: saveAllDistributions() — PENDING
    RD-->>C: số bản ghi
    C->>K: ack.acknowledge()

    Note over SCH,EXT: ══ Ranh giới thread: batch pool, MDC/trace KHÔNG qua được ══

    SCH->>M: đọc PENDING + FAILED(retry<3)
    M-->>PR: DistributionEntity[]
    PR->>M: findById(rule) + provider + template
    PR->>BS: processDistribution(context)
    BS->>CU: getCustomerProfile (nếu cần recipient/param)
    CU-->>BS: phone/email/name
    Note over BS: resolve {{voucher_code}}<br/>từ $.payload.voucher_code
    BS-->>PR: PROCESSING + message + recipient

    PR->>PS: sendMessage(ProcessedDistribution)
    PS->>EXT: JSON_REST hoặc SOAP
    EXT-->>PS: 2xx / lỗi
    PS-->>PR: ProviderResponse

    PR->>M: update status + retry + version
    PR->>K: promotion_distribution_events<br/>DISTRIBUTION_SUCCESS
```

---

## 7. Ranh giới thread và ngữ cảnh log

```mermaid
flowchart TB
    subgraph TH1["Thread: KafkaListenerEndpointContainer#N"]
        A["Consumer → dispatch → pp-coupon → Mongo"]
        A1["✅ traceId có<br/>(mdcPropagationEnabled=true)<br/>❌ eventId/ruleId/customerId KHÔNG vào MDC"]
    end

    subgraph TH2["Thread: distribution-batch-N (pool 5)"]
        B["Reader → Processor → Writer → Provider"]
        B1["❌ traceId MẤT<br/>distributionTaskExecutor() không set TaskDecorator<br/>✅ chỉ có mdc.distributionId"]
    end

    subgraph TH3["Thread: scheduler"]
        C["DistributionJobScheduler mỗi 5s"]
    end

    A -.->|"qua Mongo, KHÔNG qua lời gọi"| B
    C --> B

    style B1 fill:#fee,stroke:#c33,stroke-width:2px
    style A1 fill:#ffe,stroke:#ca3
```

Hai tầng **không nối nhau bằng lời gọi hàm** mà qua bảng `distributions`. Cộng với
việc batch pool không propagate context ⇒ **không thể trace một event từ Kafka tới
lúc tin được gửi bằng `traceId`**. Hiện chỉ nối được thủ công qua
`distributionId` / `idempotencyKey`.

---

## 8. Bảng tham số vận hành

| Tham số | Giá trị | Nguồn |
|---|---|---|
| Chu kỳ batch | **5 giây** | `DistributionJobScheduler:41` `@Scheduled(fixedRate = 5000)` |
| Bật/tắt batch | `promix.batch.enabled=true` | `application.properties:54` |
| ⚠ Property chết | `distribution.batch.enabled=false` | `:198` — **không code nào đọc**, đặt `false` không tắt được batch |
| Chunk size | 100 | `promix.batch.chunk-size` `:55` |
| Concurrency | 5 luồng | `promix.batch.concurrency-limit` `:56` |
| Skip limit | 5 | `DistributionBatchConfig` `.skipLimit(5)` |
| Retry của Spring Batch | **0** (tắt) | `.retryLimit(0)` — retry do `ProviderService` lo |
| Reader lọc retry | `< 3` | `distribution.batch.max-retries=3` `:202` |
| Provider max retry | **5** | `distribution.retry.max-attempts=5` `:212` ⚠ lệch với trên |
| Backoff | `min(2^n × 30s, 30 phút)` | `:213-214` |
| Timeout provider | 30 000 ms | `:206` |
| Timeout pp-coupon | 15 s | `distribution.coupon-service.timeout-seconds` |
| Mock mode | `false` | `:209` (success-rate 0.7 khi bật) |
| Retry on 401 | `true` | `:216` |
| Kafka AckMode | `MANUAL_IMMEDIATE` | `:141` |
| DLQ | **bật**, suffix `_dlqmo` | `:155-156` |
| Batch metadata DB | **MariaDB riêng** | `promix.batch.datasource.*` `:57-61` |
| Topic kết quả | `promotion_distribution_events` | `KafkaEventPublisher:23` |

> Batch metadata nằm ở **MariaDB** trong khi dữ liệu nghiệp vụ ở **MongoDB**.
> MariaDB chết ⇒ job không chạy ⇒ `distributions` dồn `PENDING` dù Mongo vẫn khoẻ.

---

## 9. Bản đồ điểm gãy

```mermaid
flowchart TD
    A["Event vào"] --> B{"1. Có consumer<br/>cho topic?"}
    B -- "không" --> X1["Im lặng hoàn toàn"]
    B --> C{"2. Map được<br/>type → eventCode?"}
    C -- "không" --> X2["log.warn + ack<br/>MẤT MESSAGE"]
    C --> D{"3. Có customer_id?"}
    D -- "không" --> X2
    D --> E{"4. Rule RUNNING<br/>khớp eventCode?"}
    E -- "không" --> X3["No applicable rules"]
    E --> F{"5. Condition pass?"}
    F -- "không" --> X4["Loại im lặng<br/>(log DEBUG)"]
    F --> G{"6. Idempotency?"}
    G -- "trùng" --> X5["Skip"]
    G --> H{"7. SEND_VOUCHER:<br/>phát mã OK?"}
    H -- "không" --> X6["FAILED + terminal<br/>KHÔNG BAO GIỜ gửi"]
    H --> I{"8. providerId có?"}
    I -- "không" --> X6
    I --> J{"9. Mongo save OK?"}
    J -- "không" --> X7["→ DLQ _dlqmo"]
    J --> K{"10. Batch chạy?<br/>(MariaDB sống?)"}
    K -- "không" --> X8["Kẹt PENDING vô hạn"]
    K --> L{"11. Provider/template<br/>còn active?"}
    L -- "không" --> X9["FAILED (retry được)"]
    L --> M{"12. Resolve recipient?"}
    M -- "không" --> X10["gửi tới 'unknown'"]
    M --> N{"13. Provider 2xx?"}
    N -- "không" --> X11["retry ≤5 lần<br/>rồi FAILED"]
    N --> OK["✅ SUCCESS<br/>+ Kafka event"]

    style OK fill:#efe,stroke:#393,stroke-width:3px
    style X2 fill:#fee,stroke:#c33,stroke-width:2px
    style X6 fill:#fee,stroke:#c33,stroke-width:2px
    style X8 fill:#fee,stroke:#c33,stroke-width:2px
```

| Điểm gãy | Dấu hiệu nhận biết |
|---|---|
| 1 | Không log gì cả |
| 2, 3 | `log.warn` trong consumer, **không có bản ghi nào** |
| 4 | `[DISPATCHING-VOUCHER-EVENT] No applicable rules` |
| 5 | log DEBUG `Path {} not found in payload` |
| 6 | `Idempotency hit — skip key=` |
| 7, 8 | `Distribution không gửi được — reason=` + `terminal:true` trong Mongo |
| 9 | Message nằm ở topic `<topic>_dlqmo` |
| 10 | `distributions` dồn `PENDING`, **không có** log `Writing N distribution items` |
| 11 | `Distribution X cannot be dispatched: Provider inactive/not found` |
| 12 | `Could not extract recipient ... returning unknown` |
| 13 | `Scheduling retry for distribution` rồi `Max retry attempts reached` |

---

## 10. Liên kết

- [Phần 1 — Tạo rule](01-TAO-RULE.md)
- [Phần 2 — Event nào trigger](02-EVENT-NAO-TRIGGER.md)
- [Phần 3 — Luồng dispatch](03-LUONG-DISPATCH.md)
- [Phần 4 — SEND_VOUCHER](04-ACTION-SEND-VOUCHER.md)
- [Phần 5 — SEND_NOTIFICATION](05-ACTION-SEND-NOTIFICATION.md)
- [Phần 6 — Pipeline gửi tin](06-PIPELINE-GUI-TIN.md)
- [Phần 7 — Rủi ro & checklist](07-RUI-RO-VA-CHECKLIST.md)
- Đánh giá log: [`docs-log-evaluation/`](../docs-log-evaluation/KIEM-CHUNG-V1-V9.md)
