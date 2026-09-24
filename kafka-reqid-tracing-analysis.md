# Phân tích Kafka interceptor & trace `reqId` — promix-messaging-autoconfigure

> Ngày phân tích: 2026-09-24 · Nhánh: `sync-phase2` · HEAD: `bb09ad6`
> Phạm vi: các commit từ `25a36d8` đến `bb09ad6`, module `autoconfigure/promix-messaging-autoconfigure`
> và các phần liên quan trong `promix-observer-*`, `promix-outbox-autoconfigure`.
> Phiên bản đối chiếu: Spring Boot 3.5.9 → spring-kafka 3.3.11.

---

## 1. Tóm tắt

- Module hiện có **4 lớp interceptor Kafka** (3 file, lớp `KafkaTracingInterceptor` chứa 2 inner class).
  Thực tế **chỉ 1 lớp được auto-wire mặc định**: `MdcConsumerRecordInterceptor` (phía consumer).
- `MdcProducerInterceptor` (phía producer, đọc `reqId` từ MDC) **chỉ được gắn vào `avroKafkaProducerFactory`**.
  `kafkaProducerFactory` mặc định (dùng cho JSON/DTO, tức đa số service) **không gắn interceptor này**.
- `KafkaUtils` (cách gửi được khuyến nghị) **vẫn đọc MDC key `traceId` và ghi header `correlationId`**,
  vì bean được tạo từ `KafkaProperties.TracingProperties` mà default ở đó chưa đổi (`mdcKey = "traceId"`,
  `correlationIdHeader = "correlationId"`). Hằng số mặc định mới (`reqId`) trong `KafkaUtils` chỉ áp dụng cho
  các constructor ngắn, bean auto-config không dùng chúng.
- Hệ quả: **với cấu hình mặc định, `reqId` của request HTTP không được truyền sang Kafka.** Header chứa
  hoặc Brave `traceId` (khi có Micrometer Tracing) hoặc một id `req-xxxx` sinh mới. Consumer nhận id đó làm
  `reqId`, nên chuỗi trace bị đứt ngay tại Kafka.
- Còn một số lỗi phụ: `failure()` xoá MDC trước khi error handler/DLQ chạy; `AsyncAutoConfiguration` chưa được đăng ký;
  outbox relay gửi message không có `reqId`; `LoggingMdcFilter` chỉ bật khi có `soc.system`.

---

## 2. Diễn biến các commit

| Commit | Ngày | Nội dung chính | Ảnh hưởng tới Kafka tracing |
|---|---|---|---|
| `25a36d8` | 22/09 | Thêm appender `JsonConsole` (JsonTemplateLayout, ECS) với 21 trường SOC (Phụ lục 14): `reqId`, `txnId`, `threadId`… vào `log4j2-spring.xml`. Profile non-dev dùng JSON. | Định nghĩa đầu ra log. `reqId`/`txnId` lấy từ `ctx` (ThreadContext/MDC). Pattern text của profile dev **chỉ in `traceId, spanId`**, không in `reqId`. |
| `d3e63f0` | 22/09 | Đưa hạ tầng SOC log vào `promix-observer`: `SocMdcKeys`, `@LogAction`, `SocLoggingFilter`, `SocLoggingAspect`. Kích hoạt khi có `soc.system`. | Tầng HTTP bắt đầu đặt `reqId` vào MDC (lấy từ `X-Request-ID` hoặc sinh `req-<12hex>`). |
| `9383015` | 22/09 | Bỏ tiền tố SOC (`MdcKeys`, `LoggingMdcFilter`, `LoggingAspect`), thêm enum `ActionType`/`TransactionType`/`DataType`, thêm `KafkaLoggingInterceptor` (observer). | `KafkaLoggingInterceptor` được khai báo là một bean thường, **không được gắn vào container factory** nên không có tác dụng. |
| `bb09ad6` | 24/09 | Thống nhất dùng `reqId`; gộp interceptor; thêm propagation MDC cho async. Xoá `TraceIdConsumerRecordInterceptor`, `TraceIdProducerInterceptor`, `KafkaLoggingInterceptor`. Thêm `MdcConsumerRecordInterceptor`, `MdcProducerInterceptor`, `MdcTaskDecorator`, `MdcContextExecutor`, `AsyncAutoConfiguration`. `KafkaUtils` đổi hằng số mặc định sang `reqId`. `TraceContext` chuyển sang `reqId`, các hàm `traceId` được đánh dấu deprecated. | Đây là commit chính. Thiết kế đúng hướng nhưng **phần wiring còn thiếu** (chi tiết ở mục 4). |

### Trước và sau `bb09ad6`

| | Trước | Sau |
|---|---|---|
| Consumer interceptor | `TraceIdConsumerRecordInterceptor`: đọc header `correlationId`, ghi MDC `traceId` | `MdcConsumerRecordInterceptor`: đọc `reqId > correlationId > traceId > X-Request-ID`, ghi đủ các trường Phụ lục 14 |
| Producer interceptor | `TraceIdProducerInterceptor`: **luôn sinh UUID mới** cho header `traceId` | `MdcProducerInterceptor`: lấy `reqId` từ MDC, ghi header `reqId` và `correlationId` |
| `KafkaUtils` hằng số default | header `correlationId`, MDC `traceId` | header `reqId`, MDC `reqId` (**nhưng bean auto-config không dùng hằng số này**) |
| `TracingProperties` default | `correlationId` / `traceId` | **Không đổi** ← nguyên nhân chính |

---

## 3. Danh sách interceptor Kafka trong module

| # | Lớp | Loại (API) | Được wire? | Cách wire | Vai trò thực tế |
|---|---|---|---|---|---|
| 1 | `tracing/MdcConsumerRecordInterceptor` | `RecordInterceptor` (spring-kafka) | **Có, mặc định** | Bean `mdcConsumerRecordInterceptor` trong `MessagingAutoConfiguration` (`@ConditionalOnMissingBean(RecordInterceptor.class)`, `mdc-propagation-enabled` mặc định là true) → `factory.setRecordInterceptor(...)` cho **cả** `kafkaListenerContainerFactory` và `kafkaDlqListenerContainerFactory` | Đặt MDC cho luồng `@KafkaListener` |
| 2 | `tracing/MdcProducerInterceptor` | `ProducerInterceptor` (kafka-clients) | **Chỉ với Avro** | `interceptor.classes` trong `KafkaAvroAutoConfiguration.buildAvroProducerConfig` (khi `promix.messaging.avro.enabled=true`). `MessagingAutoConfiguration.buildProducerConfig` **không** gắn. `KafkaTracingAutoConfiguration` chỉ tạo bean record chứa tên lớp (`producer-interceptor-enabled=true`), không gắn vào đâu. | Đẩy `reqId`, `userName`, `system`, `currentNode`, `message.timestamp` từ MDC vào header |
| 3 | `tracing/KafkaTracingInterceptor.Producer` | `ProducerInterceptor` | Không | Chỉ xuất hiện trong `docs/configurations/messaging/06-complete-starter-config.yml` | Code cũ: **sinh traceId/spanId ngẫu nhiên** mỗi message. Nguy hiểm nếu ai đó copy cấu hình mẫu |
| 4 | `tracing/KafkaTracingInterceptor.Consumer` | `ConsumerInterceptor` | Không | Như trên | Chỉ log debug, không đặt MDC (và `ConsumerInterceptor` chạy trên luồng poll nên cũng không đặt MDC cho listener được) |

Các thành phần liên quan không phải interceptor:
- `utils/KafkaUtils`: tự tạo header trace trước khi gọi `kafkaTemplate.send(...)`.
- `KafkaTemplate.setObservationEnabled(true)` và `ContainerProperties.setObservationEnabled(true)`: nếu có Micrometer Tracing (observer có `micrometer-tracing-bridge-brave`), Spring Kafka tự inject/extract header `b3`/`traceparent` để nối **Brave traceId** (phục vụ APM). Cơ chế này tách biệt với `reqId` nghiệp vụ.

---

## 4. Cách hoạt động hiện tại

### 4.1 Luồng mong muốn (theo Javadoc của commit)

```
HTTP (X-Request-ID) ──► LoggingMdcFilter: MDC.reqId
                               │
                               ▼
               KafkaUtils / KafkaTemplate.send
                               │  header reqId (+ correlationId)
                               ▼
               MdcConsumerRecordInterceptor: MDC.reqId = header.reqId
                               │
                               ▼
                    @KafkaListener → log có cùng reqId
```

### 4.2 Luồng thực tế với cấu hình mặc định (JSON, không bật Avro)

```
HTTP ──► TracingFilter:      MDC.traceId = <Brave traceId>  (nếu có Tracer)
    └──► LoggingMdcFilter:   MDC.reqId   = X-Request-ID | req-xxxx   (chỉ khi có soc.system)

Service gọi kafkaUtils.send(topic, key, dto)
   KafkaUtils bean: correlationIdHeader="correlationId", mdcKey="traceId"   ← từ TracingProperties
   getOrGenerateCorrelationId():  MDC.get("traceId")
        ├─ có Brave  → header correlationId = <Brave traceId>   (≠ reqId)
        └─ không có  → header correlationId = req-<random>      (≠ reqId)
   (correlationIdHeader == "correlationId" nên không thêm header trùng)

KafkaProducer.send ─► interceptor.classes: (trống) → MdcProducerInterceptor KHÔNG chạy
                   → message KHÔNG có header "reqId"

Consumer: MdcConsumerRecordInterceptor
   reqId header? không có → correlationId header? có → MDC.reqId = <Brave traceId | req-random>
   ⇒ reqId ở consumer KHÁC reqId ở producer → không trace được bằng reqId
```

Nếu gửi thẳng bằng `kafkaTemplate.send(...)` (không qua `KafkaUtils`), ví dụ `KafkaOutboxEventPublisher`,
message **không có header trace nào**. Consumer sẽ sinh `kafka-<8hex>`.

### 4.3 Luồng khi bật Avro (`promix.messaging.avro.enabled=true`)

`avroKafkaProducerFactory` có `interceptor.classes=MdcProducerInterceptor`.
`KafkaUtils.createHeaders(...)` (SpecificRecord) ghi header `correlationId` (giá trị sai như 4.2), sau đó
`MdcProducerInterceptor` thêm header `reqId` lấy từ `MDC.reqId` (đúng). Consumer ưu tiên `reqId` nên **trace đúng**,
nhưng header `correlationId` và `reqId` mang hai giá trị khác nhau, dễ gây nhầm lẫn.

> Lưu ý: `avroKafkaProducerFactory` là bean kiểu `ProducerFactory` được `@Import` trước `MessagingAutoConfiguration`,
> nên có thể khiến `kafkaProducerFactory` (`@ConditionalOnMissingBean`) bị bỏ qua khi bật Avro. Cần xác minh bằng
> `--debug` / conditions report nếu có service bật Avro.

### 4.4 Chi tiết `MdcConsumerRecordInterceptor` (spring-kafka 3.3.11)

Thứ tự gọi (đã kiểm tra bytecode `KafkaMessageListenerContainer$ListenerConsumer`):

```
intercept()                      → đặt MDC (threadId, reqId, txnId mới, parentNode=topic:partition, ...)
listener.onMessage()
  ├─ OK   → success()  (dùng default, không làm gì)
  └─ Lỗi  → failure()  → invokeErrorHandler() (retry / DLQ / log lỗi)
afterRecord()                    → endTime, duration, MDC.clear()
```

- `failure()` hiện **tự gọi `afterRecord()`**, dẫn tới `MDC.clear()` chạy **trước** `invokeErrorHandler`.
  Log của `DefaultErrorHandler`, `CommittingErrorHandler`, `EnhancedDeadLetterPublishingRecoverer` và lambda log lỗi
  trong `kafkaCommonErrorHandler` **không còn `reqId`**, đúng lúc cần trace nhất. Ngoài ra `afterRecord` bị gọi 2 lần.
- `txnType` = `"100"` cho consumer, trong khi `LoggingMdcFilter` dùng `"01"` và enum `TransactionType` có SIMPLE/PARENT/CHILD.
  Nên lấy giá trị từ enum `TransactionType.CHILD` cho thống nhất (enum ở observer-core, cần cân nhắc dependency).
- `RecordInterceptor` **không chạy cho batch listener** (`promix.messaging.kafka.consumer.batch-listener=true`).
  Trường hợp này cần `BatchInterceptor`.
- Interceptor luôn bật, không phụ thuộc `soc.system`, còn filter HTTP thì phụ thuộc. Hành vi giữa HTTP và Kafka vì thế không đồng nhất.

### 4.5 Chi tiết `MdcProducerInterceptor`

- Chạy trên **luồng gọi `send()`** (kafka-clients gọi `onSend` đồng bộ trong `KafkaProducer.send`), nên đọc MDC được.
  Đây là chỗ đúng để lấy `reqId`.
- Chỉ `addHeaderIfAbsent`, nên header do code ứng dụng/`KafkaUtils` đặt sẽ được ưu tiên.
- Key cấu hình `promix.messaging.tracing.producer-interceptor-enabled` **không được set ở đâu**. Còn
  `KafkaAvroAutoConfiguration` truyền `trace-id-header-enabled` / `span-id-header-enabled` (key của interceptor cũ),
  mà interceptor mới bỏ qua các key này.
- Khi MDC không có `reqId` (scheduler, outbox relay, luồng async không propagate), interceptor sinh `req-<12hex>`.
  Cách này chấp nhận được với message gốc, nhưng nó che mất lỗi mất context.

### 4.6 Phía observer liên quan

| Thành phần | Trạng thái |
|---|---|
| `LoggingMdcFilter` | Đặt `reqId` từ `X-Request-ID` (hoặc sinh mới). **Chỉ bật khi có `soc.system`.** Không trả `X-Request-ID` về response. Dùng `ThreadContext` (log4j2); cùng backing map với SLF4J `MDC` vì starter dùng `spring-boot-starter-log4j2`. |
| `TracingFilter` | Đặt `MDC.traceId`/`spanId` = Brave. Đây chính là giá trị mà `KafkaUtils` đang lấy nhầm. |
| `AsyncAutoConfiguration` | **Không được đăng ký**: không có trong `AutoConfiguration.imports` của observer, cũng không nằm trong `@Import` của `PromixObserverAutoConfiguration`. Hiện là dead code. Nếu đăng ký nguyên trạng, bean `Executor taskExecutor` sẽ tắt `applicationTaskExecutor` mặc định của Boot. |
| `MdcTaskDecorator` / `MdcContextExecutor` | Code ổn (capture, restore rồi khôi phục context cũ), nhưng chưa được dùng ở đâu. |
| `log4j2-spring.xml` profile dev | Pattern text chỉ có `[%X{traceId}, %X{spanId}]`, **không in `reqId`**, nên khó kiểm thử local. |
| Feign `CommonRequestInterceptor` | `getCurrentCorrelationId()` luôn trả `null`, gửi UUID ngẫu nhiên. Chuỗi HTTP→HTTP cũng chưa truyền `reqId` (ngoài phạm vi Kafka nhưng cùng mục tiêu). |

### 4.7 Outbox

`KafkaOutboxEventPublisher` chạy trên luồng relay/scheduler (MDC rỗng) và gọi thẳng `kafkaTemplate.send`.
Metadata của event được đẩy thành header có tiền tố `X-Meta-<key>`. Consumer không đọc `X-Meta-reqId`, nên dù có lưu
`reqId` vào metadata thì trace vẫn đứt. Muốn trace xuyên outbox phải **lưu `reqId` lúc ghi outbox (luồng HTTP)**
và **publish lại dưới header `reqId`**.

---

## 5. Danh sách vấn đề

| # | Mức độ | Vấn đề | Vị trí |
|---|---|---|---|
| P1 | **Cao** | `KafkaUtils` bean đọc MDC `traceId` và ghi header `correlationId` do default của `TracingProperties` chưa đổi. `reqId` không được truyền. | `core/promix-messaging-core/.../KafkaProperties.java:169,176`; `utils/KafkaUtilsAutoConfiguration.java:33-39` |
| P2 | **Cao** | `MdcProducerInterceptor` không được gắn vào `kafkaProducerFactory` mặc định (JSON). | `MessagingAutoConfiguration.java:185-235` |
| P3 | **Cao** | Outbox publish không mang `reqId` (MDC rỗng, metadata có tiền tố `X-Meta-`). | `promix-outbox-autoconfigure/.../KafkaOutboxEventPublisher.java:95-113` |
| P4 | Trung bình | `failure()` gọi `afterRecord()` nên MDC bị clear trước error handler/DLQ, và `afterRecord` chạy 2 lần. | `MdcConsumerRecordInterceptor.java:150-161` |
| P5 | Trung bình | `AsyncAutoConfiguration` không được đăng ký. `@Async`/executor không propagate `reqId`. Kafka gửi từ luồng async sẽ sinh id mới. | observer `AutoConfiguration.imports`, `PromixObserverAutoConfiguration` |
| P6 | Trung bình | `LoggingMdcFilter` chỉ bật khi có `soc.system`. Service không khai báo sẽ không có `reqId` ở HTTP. | `LoggingAutoConfiguration.java:48-60` |
| P7 | Thấp | Header `correlationId` và `reqId` có thể mang 2 giá trị khác nhau (luồng Avro). | `KafkaUtils.createHeaders/createGenericHeaders` |
| P8 | Thấp | `createHeaders`/`createGenericHeaders` (Avro) không thêm header `reqId` như `createJsonHeaders`. Logic tạo header trace bị lặp ở 3 nơi. | `KafkaUtils.java:449-545` |
| P9 | Thấp | Code/tài liệu cũ: `KafkaTracingInterceptor` (id ngẫu nhiên), Javadoc `KafkaTracingAutoConfiguration` vẫn nói `mdc-key: traceId` và `TraceIdConsumerRecordInterceptor`, `docs/.../06-complete-starter-config.yml` trỏ tới `KafkaTracingInterceptor$Producer/Consumer`. | `tracing/`, `docs/configurations/messaging/` |
| P10 | Thấp | `spring.factories` có dòng hỏng (`...AvroSerializationConfigcom.promix...`) trỏ tới lớp không tồn tại. Boot 3 bỏ qua key `EnableAutoConfiguration` trong file này nên hiện vô hại, nhưng nên xoá. | `META-INF/spring.factories` |
| P11 | Thấp | Batch listener không có MDC (thiếu `BatchInterceptor`). | `MessagingAutoConfiguration.kafkaListenerContainerFactory` |
| P12 | Thấp | Pattern log dev không in `reqId`. | `log4j2-spring.xml:14,19` |
| P13 | Thấp | Module messaging không có test (`src/test` trống) cho interceptor/wiring. | — |

---

## 6. Đề xuất sửa (theo thứ tự ưu tiên)

### 6.1 Đổi default `TracingProperties` sang `reqId` (sửa P1)

```java
// core/promix-messaging-core/.../KafkaProperties.java  (TracingProperties)
/** Header chứa business request id khi gửi/nhận. */
private String correlationIdHeader = "reqId";
/** MDC key chứa business request id. */
private String mdcKey = "reqId";
```

`KafkaUtils.createJsonHeaders` đã tự thêm `correlationId` để tương thích ngược khi header khác `correlationId`.
Service nào cấu hình đè `promix.messaging.kafka.tracing.mdc-key` / `correlation-id-header` cần rà lại.

### 6.2 Gắn `MdcProducerInterceptor` vào producer factory mặc định (sửa P2)

Nên gộp với `interceptor.classes` mà người dùng tự khai báo, không ghi đè:

```java
// MessagingAutoConfiguration.buildProducerConfig(...) — trước dòng putAll(producer.getProperties())
if (kafkaProperties.getTracing().isEnabled()) {
    configProps.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, MdcProducerInterceptor.class.getName());
}
configProps.putAll(kafkaProperties.getProducer().getProperties());
// sau putAll: nếu user có interceptor.classes riêng thì nối thêm MdcProducerInterceptor
mergeInterceptor(configProps, MdcProducerInterceptor.class.getName());
```

Nên làm tương tự trong `KafkaAvroAutoConfiguration`, đồng thời bỏ 2 key `trace-id-header-enabled` / `span-id-header-enabled`
không còn tác dụng. Có thể bỏ luôn bean `TracingProducerInterceptorClassName` vì không còn cần thiết.

Khi đã có P1 và P2, `KafkaTemplate.send` trực tiếp cũng có `reqId`. `KafkaUtils` và interceptor cùng đọc `MDC.reqId`
nên không còn lệch giá trị.

### 6.3 Sửa `failure()` (sửa P4)

```java
@Override
public void failure(ConsumerRecord<String, Object> record, Exception exception,
                    Consumer<String, Object> consumer) {
    MDC.put(STATUS, "1");
    MDC.put(ERROR_CODE, "9999");
    if (exception != null) { /* errDesc như cũ */ }
    // KHÔNG gọi afterRecord(): spring-kafka tự gọi afterRecord() sau khi error handler chạy xong
}
```

### 6.4 Outbox mang `reqId` (sửa P3)

1. Khi ghi outbox (luồng HTTP/listener), đưa `MDC.get("reqId")` vào `metadata` của event (ví dụ key `reqId`).
2. Trong `KafkaOutboxEventPublisher`, nếu metadata có `reqId` thì `addHeader(record, "reqId", ...)` (không có tiền tố `X-Meta-`).
   Có thể thêm cả `correlationId`. Cách khác: publisher `MDC.put("reqId", ...)` quanh lệnh `send` để interceptor tự lấy,
   sau đó `MDC.remove` trong `finally`.

### 6.5 Propagation async (sửa P5)

Nên tránh tự định nghĩa `Executor taskExecutor`. Chỉ cần đăng ký `TaskDecorator`: Boot 3.x
`TaskExecutionAutoConfiguration` sẽ tự áp dụng nó cho `applicationTaskExecutor` (dùng cho `@Async`).

```java
@Bean
@ConditionalOnMissingBean(TaskDecorator.class)
public TaskDecorator mdcTaskDecorator() { return new MdcTaskDecorator(); }
```

Sau đó đăng ký cấu hình (thêm vào `AutoConfiguration.imports` hoặc `@Import` trong `PromixObserverAutoConfiguration`).
Nếu dự án dùng Micrometer context propagation thì có thể kết hợp với `ContextPropagatingTaskDecorator`.

### 6.6 Các việc dọn dẹp

- P6: cho `LoggingMdcFilter` (ít nhất phần `reqId`) chạy cả khi thiếu `soc.system`, hoặc ghi rõ yêu cầu này trong README.
  Nên trả `X-Request-ID` trong response.
- P7/P8: gom logic tạo header trace của `KafkaUtils` vào một hàm dùng chung. Avro cũng ghi `reqId` và `correlationId` cùng giá trị.
- P9/P10: xoá `KafkaTracingInterceptor`, sửa Javadoc `KafkaTracingAutoConfiguration`, sửa docs mẫu, xoá dòng hỏng trong `spring.factories`.
- P11: thêm `BatchInterceptor` đặt `reqId` theo record đầu tiên, hoặc hướng dẫn set MDC thủ công trong batch listener.
- P12: thêm `%X{reqId}` vào pattern dev.
- Feign: override `getCurrentCorrelationId()` để trả `MDC.get("reqId")` và gửi kèm `X-Request-ID`.

---

## 7. Checklist kiểm thử đề xuất

1. **Unit** `MdcProducerInterceptor`: MDC có `reqId`, header `reqId`/`correlationId` đúng giá trị. Header đã có sẵn thì không bị ghi đè.
2. **Unit** `MdcConsumerRecordInterceptor`: thứ tự ưu tiên header. Sau `failure()` MDC **vẫn còn** `reqId`. Sau `afterRecord()` MDC rỗng.
3. **Wiring test** (`ApplicationContextRunner`): `kafkaProducerFactory.getConfigurationProperties()` chứa
   `interceptor.classes` có `MdcProducerInterceptor`. Trường hợp user có sẵn interceptor thì được nối thêm.
   Bean `KafkaUtils` có `mdcKey == "reqId"`.
4. **Integration** (`spring-kafka-test` / EmbeddedKafka): gọi HTTP với `X-Request-ID: req-abc`, service gửi Kafka,
   listener log ra `reqId=req-abc`. Lặp lại với listener ném exception: log của error handler/DLQ vẫn có `req-abc`,
   message DLQ giữ header `reqId`.
5. **Outbox**: ghi event trong request `req-abc`, relay publish, consumer nhận `reqId=req-abc`.
6. **Async**: `@Async` method gửi Kafka vẫn giữ `reqId`.
