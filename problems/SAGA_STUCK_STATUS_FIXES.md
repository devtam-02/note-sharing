# Báo cáo xử lý sự cố: Campaign kẹt trạng thái

**Phạm vi:** service `promotion-campaign` — luồng tạo campaign (create), kích hoạt theo lịch (activate) và bật cashback (enable)
**Thời gian điều tra:** 13–14/08/2026, môi trường UAT
**Nhánh:** `tamntt25/putting-log-for-scheduler`

---

## 1. Tóm tắt cho người không chuyên

### Chuyện gì đã xảy ra

Người dùng tạo một chiến dịch cashback. Đáng lẽ chiến dịch phải tự động đi qua các bước và cuối cùng ở trạng thái **đang chạy** (`RUNNING`). Nhưng thực tế nó **đứng im** ở một trạng thái trung gian — lúc thì `INITIALIZING` (đang khởi tạo), lúc thì `ENABLING` (đang bật) — và **nằm đó vĩnh viễn**, không báo lỗi, không tự phục hồi.

Nhìn từ phía người dùng: bấm tạo → thấy báo thành công → nhưng chiến dịch không bao giờ chạy, và cũng không có thông báo lỗi nào.

### Hình dung bằng ví dụ

Hãy tưởng tượng chiến dịch như **một hồ sơ đi qua dây chuyền nhiều bộ phận**. Mỗi bộ phận làm xong thì bấm nút "chuyển sang bộ phận kế tiếp" trên một bảng điều khiển. Bảng này chỉ chấp nhận nút bấm đúng thứ tự.

Chúng tôi tìm ra **ba kiểu sự cố**, đều thuộc cùng một họ vấn đề:

**Kiểu 1 — Bấm nút quá sớm**
Nhân viên bấm "chuyển bước tiếp theo" trong khi bảng điều khiển vẫn đang ghi nhận việc hồ sơ vừa vào bộ phận hiện tại. Bảng từ chối cú bấm đó. Tệ hơn: **hệ thống không hề ghi lại việc bị từ chối**, nên hồ sơ nằm im mà không ai biết vì sao.

**Kiểu 2 — Đọc sổ trước khi sổ được đóng**
Bộ phận A ghi vào sổ "hồ sơ này đang xử lý" nhưng **chưa đóng sổ**. Cùng lúc, bộ phận B mở sổ ra xem, vẫn thấy dòng cũ, tưởng chưa tới lượt mình nên bỏ qua và đi làm việc khác. Vài mili giây sau bộ phận A mới đóng sổ — nhưng đã muộn, không còn ai quay lại xử lý hồ sơ nữa.

**Kiểu 3 — Huỷ quy trình nhưng quên cập nhật hồ sơ**
Khi hệ thống phát hiện một hồ sơ bị treo quá lâu, nó đánh dấu "quy trình đã huỷ" trong sổ nội bộ — nhưng **quên đổi trạng thái trên chính hồ sơ**. Kết quả: hồ sơ vẫn hiện "đang xử lý" với người dùng, mãi mãi.

### Vì sao lúc test ở máy dev thì không thấy lỗi

Vì kiểu 1 và kiểu 2 là **cuộc đua thời gian**, không phải lỗi logic. Ai nhanh hơn vài mili giây thì thắng.

Ở máy dev, cơ sở dữ liệu nằm ngay trên máy nên mọi thao tác ghi cực nhanh — hệ thống luôn thắng cuộc đua với biên khoảng **20–30 mili giây**. Trên UAT, cơ sở dữ liệu nằm ở máy chủ riêng, mỗi lần ghi chậm hơn khoảng 9 lần — hệ thống thua cuộc đua với biên khoảng **6–12 mili giây**.

**Cùng một đoạn mã, không sửa gì cả:** dev luôn đúng, UAT luôn sai. Và quan trọng: **dev không hề "an toàn"** — biên thắng chỉ 20ms, chỉ cần máy bận một chút là dev cũng hỏng y hệt.

### Đã xử lý thế nào

Chúng tôi không cố "làm cho nhanh hơn" — đó vẫn là đánh cược vào thời gian. Thay vào đó sửa tận gốc:

1. **Bỏ hẳn những bước bấm nút thừa.** Các bước trung gian không làm việc gì thực chất, chỉ để bấm nút chuyển tiếp — đã được gỡ bỏ. Không còn nút để bấm sai thời điểm.
2. **Bắt buộc đóng sổ trước khi báo cho bộ phận sau.** Giờ hệ thống chỉ gửi yêu cầu đi sau khi dữ liệu đã được ghi chắc chắn.
3. **Khi huỷ quy trình thì cập nhật luôn hồ sơ** sang trạng thái `ERROR`, để người dùng nhìn thấy có sự cố thay vì chờ vô tận.
4. **Bổ sung nhật ký chẩn đoán** để lần sau sự cố tương tự lộ ra ngay thay vì im lặng.

---

## 2. Bảng tổng hợp

| # | Vấn đề | Triệu chứng | Trạng thái |
|---|---|---|---|
| 1 | Event bị từ chối trong saga **enable** | Campaign kẹt `ENABLING` | Đã sửa, đã deploy, **đã xác nhận hết** |
| 2 | Saga huỷ nhưng không trả lại trạng thái campaign | Campaign kẹt `ENABLING` | Đã sửa, đã deploy, **đã xác nhận chạy** |
| 3 | Đọc dữ liệu trước khi transaction commit | Campaign kẹt `ENABLING` | Đã sửa, đã deploy, **chờ xác nhận** |
| 4 | Event bị từ chối trong saga **create** | Campaign kẹt `INITIALIZING` | Đã sửa, **chưa deploy** |
| 5 | Timeout huỷ saga nhưng không đổi trạng thái campaign | Campaign kẹt im lặng, không báo lỗi | Đã sửa, **chưa deploy** |
| 6 | Nhật ký thiếu / sai, che mất nguyên nhân | Không chẩn đoán được | Đã sửa, đã deploy |

> **Lưu ý:** toàn bộ thay đổi chưa được biên dịch trong môi trường phân tích (không có `java`/`mvn`). Cần build lại trước khi deploy.

---

## 3. Vì sao sandbox chạy đúng mà UAT lại sai

Đây là câu hỏi quan trọng nhất, vì nó giải thích tại sao lỗi lọt qua khâu test.

Lấy mốc 0 là thời điểm ghi trạng thái mới xuống cơ sở dữ liệu:

| Mốc | Sandbox lần 1 | Sandbox lần 2 | UAT lần 1 | UAT lần 2 |
|---|---:|---:|---:|---:|
| Gửi lệnh sang service cashback | +20ms | +19ms | +157ms | +169ms |
| Nhận phản hồi | +57ms | +49ms | +264ms | +263ms |
| Đọc lại dữ liệu | +68ms | +53ms | +319ms | +319ms |
| **Transaction commit** | **+37ms** | **+31ms** | **+331ms** | **+325ms** |
| **Biên an toàn** | **+31ms** ✅ | **+22ms** ✅ | **−12ms** ❌ | **−6ms** ❌ |

Điều kiện chạy đúng: **commit phải xảy ra trước lần đọc lại**.

Nguyên nhân chênh lệch nằm ở phần việc *bên trong* transaction — toàn bộ là thao tác ghi cơ sở dữ liệu:

| Đoạn | Sandbox | UAT | Chậm hơn |
|---|---:|---:|---:|
| Ghi nhật ký saga (`saga_execution_logs`) | 3ms | 21ms | 7× |
| Ghi trạng thái máy (`state_machine`) | ~25ms | ~223ms | ~9× |
| **Tổng thời gian transaction** | **37ms** | **331ms** | **8.9×** |
| Round-trip Kafka sang cashback | 37ms | 107ms | 2.9× |

Cơ sở dữ liệu sandbox là `localhost` (xem `application-dev.yml`), UAT là máy chủ riêng. Phần việc *trong* transaction chậm đi ~9 lần, còn phần *chờ phản hồi* chỉ chậm ~3 lần. Nên UAT làm vế "chưa commit" dài ra nhanh hơn vế "chờ phản hồi" — và thứ tự bị đảo.

**Bốn lần chạy, biên nằm trong khoảng −12ms đến +31ms.** Kết quả đúng/sai lật theo dấu của con số này. Đó là đặc trưng của race condition: kết quả do vài mili giây quyết định, không do logic.

---

## 4. Chi tiết từng vấn đề

### Vấn đề 1 — Event bị từ chối trong saga enable

**Hiện tượng:** campaign kẹt `ENABLING`; log lặp lại chu kỳ ~36 giây rồi bị timeout monitor huỷ.

**Nguyên nhân:** `CashbackEnableCampaignAction` là một *state action* — Spring StateMachine chạy nó **bất đồng bộ trên luồng riêng**, song song với chính transition vừa đưa máy vào state đó. Action này không làm gì cả, chỉ gửi ngay event kế tiếp ngược vào máy (trong 7ms). Event tới khi transition trước chưa hoàn tất → bị trả về `DENIED`. Spring StateMachine **không xếp lại hàng đợi** event đã bị từ chối, nên saga đứng im vĩnh viễn.

**Bằng chứng (log UAT 13/08):**
```
14:08:53.236  Execute CashbackEnableCampaignAction              (luồng parallel-1)
14:08:53.348  Event ENABLE_CASHBACK_CAMPAIGN_SUCCESS → DENIED
14:08:53.498  Saga started ... currentState=CASHBACK_CAMPAIGN_ENABLE_PENDING
                             ↑ transition mới xong, muộn hơn 150ms
```

Đối chứng: saga *create* dùng đúng pattern này nhưng chạy được, vì action của nó mất vài giây (gọi Kafka rồi chờ) — lúc gửi event thì transition đã xong từ lâu. Thuần tuý là vấn đề thời gian.

**Cách xử lý:** bỏ hẳn state trung gian. Action đó chỉ chuyển tiếp, không có giá trị nghiệp vụ.

```
Trước:  START_ENABLE → CASHBACK_CAMPAIGN_ENABLE_PENDING → CASHBACK_ENABLE_PENDING
Sau:    START_ENABLE → CASHBACK_ENABLE_PENDING
```

**File:**
- `application/SagaOrchestrationEnableCashbackConfiguration.java` — gộp state
- `application/action/enable/CashbackEnableCampaignAction.java` — **đã xoá**
- `application/action/enable/CompensateEnableCashback.java` — **đã xoá** (cũng tự gửi event, sẽ dính lỗi tương tự khi DB chậm hơn chút nữa)

**Kết quả:** log UAT sau khi deploy không còn `DENIED` ở luồng enable.

---

### Vấn đề 2 — Saga huỷ nhưng không trả lại trạng thái campaign

**Hiện tượng:** khi saga enable thất bại, campaign nằm lại `ENABLING` thay vì quay về trạng thái cũ.

**Nguyên nhân:** `CompensateEnableCashback` chỉ hoàn tác khi campaign đang ở `RUNNING`. Nhưng đường đi từ scheduler để campaign ở `ENABLING`, nên nó chỉ ghi một dòng cảnh báo *"expected RUNNING for compensation"* rồi bỏ qua. Ngoài ra, nhánh huỷ trực tiếp `... → SAGA_ABORTED` **không gắn hành động hoàn tác nào**.

**Cách xử lý:**
- Thêm `RevertCampaignFromEnablingAction`: đưa campaign khỏi `ENABLING`/`RUNNING` về đúng trạng thái trước đó.
- Truyền `previousStatus` vào ngữ cảnh saga từ **cả hai** điểm khởi động (scheduler và API enable thủ công), thay vì mặc định cứng về `DISABLED`. Nhờ vậy campaign được scheduler kích hoạt sẽ quay về `ACTIVE` chứ không bị đẩy sai sang `DISABLED`.
- Gắn hành động này lên transition huỷ.

**File:**
- `application/action/enable/RevertCampaignFromEnablingAction.java` — **file mới**
- `application/SagaOrchestrationEnableCashbackConfiguration.java` — gắn vào transition
- `application/service/scheduler/CampaignActivationSchedulerService.java` — thêm `previousStatus`
- `application/service/campaign/cashback/EnableCashbackCampaignService.java` — thêm `previousStatus`

**Kết quả:** đã xác nhận chạy trên UAT — `13:04:30.512 : Reverted campaign status: ENABLING -> DISABLED`.

---

### Vấn đề 3 — Đọc dữ liệu trước khi transaction commit

**Hiện tượng:** saga enable chạy **thành công** hoàn toàn, nhưng campaign vẫn kẹt `ENABLING`. Log có dòng: *"Campaign ... is in unexpected state: ACTIVE, expected ENABLING or RUNNING"*.

**Nguyên nhân:** `processAction` được đánh dấu `@Transactional(REQUIRES_NEW)` và bọc **cả hai** việc:

1. Ghi trạng thái campaign thành `ENABLING`
2. Khởi động saga → gửi lệnh Kafka sang service cashback

Tức là lệnh được **gửi ra ngoài trong khi transaction còn mở**. Service cashback trả lời sau 92–107ms, nhanh hơn thời gian transaction còn sống (~330ms). Khi `UpdateCampaignToRunningAction` xử lý phản hồi, nó chạy trên **luồng Kafka, trong transaction riêng**, đọc cơ sở dữ liệu và vẫn thấy giá trị cũ là `ACTIVE`. Nó rơi vào nhánh "trạng thái không mong đợi", **không làm gì cả**, rồi transaction của scheduler mới commit `ENABLING`.

**Bằng chứng (UAT 14/08, hai lần tái hiện độc lập):**
```
04:52:41.655  save ENABLING (trong transaction, chưa commit)
04:52:41.919  nhận phản hồi từ cashback service
04:52:41.974  đọc DB → thấy ACTIVE   ✗
04:52:41.980  transaction commit     ← muộn hơn 6ms
```

Chi tiết làm vấn đề tệ hơn con số: MariaDB mặc định `REPEATABLE READ`, nên transaction đọc chốt ảnh chụp dữ liệu ngay lúc **mở** (`.956`) chứ không phải lúc đọc (`.974`). Cửa sổ thua thực tế là 24ms.

**Cách xử lý:** hoãn việc khởi động saga sang **sau khi transaction commit**, dùng `TransactionSynchronization.afterCommit`. Sửa ở một chỗ duy nhất trong orchestrator nên bao được cả đường scheduler lẫn đường API thủ công (đường thủ công dính đúng lỗi này, chỉ chưa lộ vì trước đó saga đã chết ở vấn đề 1).

Sau khi sửa, thứ tự luôn là: ghi `ENABLING` → **commit** → gửi lệnh → nhận phản hồi. Quan hệ thời gian biến thành quan hệ nhân quả, không còn biên nào để thua.

Kèm theo: nếu saga khởi động thất bại sau commit thì không thể ném lỗi ngược về người gọi được nữa, nên orchestrator tự hoàn tác trạng thái campaign — nếu không nó sẽ nằm `ENABLING` mà chẳng có saga nào chạy.

**File:**
- `application/CampaignSagaEnableOrchestrator.java` — hoãn sang `afterCommit`, xử lý lỗi sau commit
- `application/action/enable/RevertCampaignFromEnablingAction.java` — cho phép gọi từ ngoài state machine
- `application/action/enable/UpdateCampaignToRunningAction.java` — nâng cảnh báo lên `ERROR` kèm chữ "stranded"

---

### Vấn đề 4 — Event bị từ chối trong saga create

**Hiện tượng:** campaign kẹt `INITIALIZING` ngay sau khi tạo, chưa hề chạm tới scheduler.

**Nguyên nhân:** **giống hệt vấn đề 1**, nhưng ở saga create. `StartCashbackCampaignAction` là state action trên `START_CREATE`, không làm gì ngoài việc gửi `START_SAGA_CREATE` ngược vào chính máy đang chạy nó. Tệ hơn, nó còn đẩy qua một luồng ảo nữa, và dùng `.subscribe()` **trần không nhận kết quả** — nên khi bị `DENIED`, không có một dòng log nào để lần ra.

**Bằng chứng (UAT 14/08, sau khi thêm nhật ký chẩn đoán):**
```
07:25:06.291  CREATE-07  sending START_SAGA_CREATE, currentState=START_CREATE
07:25:06.404  CREATE-08  START_SAGA_CREATE resultType=DENIED
07:25:06.413             Saga started ... currentState=START_CREATE
                                      ↑ transition mới xong, muộn hơn 9ms
```

Một chi tiết đáng chú ý: log in ra `currentState=START_CREATE` — máy **đã** báo đúng trạng thái rồi mà event vẫn bị từ chối. Nghĩa là nguyên nhân không phải "sai trạng thái" mà là "máy còn đang bận hoàn tất transition trước".

**Cách xử lý:** bỏ state trung gian, giống vấn đề 1.

```
Trước:  CASHBACK_CAMPAIGN_INITIALIZING → START_CREATE → VALIDATION_RULE_ASSIGNING
Sau:    CASHBACK_CAMPAIGN_INITIALIZING → VALIDATION_RULE_ASSIGNING
```

**File:**
- `application/SagaOrchestrationCreateConfiguration.java` — gộp state
- `application/action/create/StartCashbackCampaignAction.java` — **đã xoá**

---

### Vấn đề 5 — Timeout huỷ saga nhưng không đổi trạng thái campaign

**Hiện tượng:** campaign kẹt trạng thái trung gian mà **không có bất kỳ dấu hiệu lỗi nào** trên giao diện.

**Nguyên nhân:** `SagaTimeoutMonitor` có ba nhánh "huỷ cưỡng bức" chỉ ghi bảng `state_machine` thành `SAGA_ABORTED` mà **không đụng gì tới trạng thái campaign**:

| Tình huống |
|---|
| Bước hoàn tác quá hạn |
| Saga update không có sự kiện lỗi tương ứng |
| Saga khác không có sự kiện lỗi tương ứng ← chính là ca `START_CREATE` |

Log thể hiện rõ:
```
07:25:55.714  No failure event mapping for state: START_CREATE
07:25:55.714  ... marking saga ... as SAGA_ABORTED
```
Saga bị đánh dấu huỷ, nhưng campaign vẫn `INITIALIZING`. **Đây chính là lý do sự cố im lặng suốt.**

**Cách xử lý:** thêm `failCampaign(...)` gọi ở cả ba nhánh — đưa campaign từ trạng thái đang xử lý (`INITIALIZING`, `ENABLING`, `DISABLING`, `UPDATING`) sang `ERROR`, kèm lý do và mốc thời gian trong metadata theo đúng định dạng đang dùng sẵn.

Một số điểm cẩn trọng đã tính tới:
- Chỉ đổi khi campaign đang ở trạng thái *đang xử lý*. Nếu nó đã `ACTIVE`/`RUNNING`/`ERROR` thì để yên, tránh việc một timeout đến muộn ghi đè lên kết quả đúng.
- Lỗi trong `failCampaign` được bắt và ghi log — một campaign hỏng không được phép chặn vòng quét của các saga còn lại.

**File:** `application/service/saga/SagaTimeoutMonitor.java`

---

### Vấn đề 6 — Nhật ký thiếu và sai, che mất nguyên nhân

Sự cố kéo dài chủ yếu vì **hệ thống không để lại dấu vết**. Các điểm mù đã được xử lý:

| Điểm mù | Hệ quả | Xử lý |
|---|---|---|
| Kết quả gửi event ghi ở mức `debug` | Mọi `DENIED` trong saga create đều vô hình ở UAT | Nâng lên `info`, kèm `resultType` |
| `.subscribe()` trần trong action create | `DENIED` không để lại dấu vết nào | Ghi lại kết quả |
| Spring StateMachine nuốt lỗi của action | Action ném lỗi thì saga chết im lặng | Ghi log lỗi trong action + override `stateMachineError` |
| `updateCampaignStatus` đọc trạng thái cũ **sau** khi đã gán mới | Luôn in `oldStatus == newStatus`, che mất trạng thái thật | Chụp giá trị trước khi gán |
| `scheduleCampaignActivation` nuốt stacktrace | Không biết vì sao lập lịch thất bại | Truyền exception vào log |
| Log tham số bị hoán vị trong `activateCampaign` | In ra `"Campaign [ACTIVE, DISABLED...] is not in <id> status"` | Sửa thứ tự |
| Không nhìn thấy thời điểm commit | Không phân biệt được lỗi logic với race condition | Thêm `CampaignTrace.logOnCommit` |

**File mới:** `application/service/saga/CampaignTrace.java`

---

## 5. Công cụ chẩn đoán: `CampaignTrace`

Toàn bộ luồng từ API tạo campaign đến lúc bật cashback đã được gắn mốc theo định dạng:

```
[<campaignId>] <MÃ-BƯỚC> | thread=<luồng> | tx=<trạng thái transaction> | <nội dung>
```

Lọc cả luồng bằng một lệnh:
```bash
grep '<campaignId>' application.log
```

Trường `tx=` là thứ quan trọng nhất: `tx=ACTIVE(...)` nghĩa là dữ liệu ghi ở dòng đó **chưa nhìn thấy được** từ luồng khác.

### Các mốc

| Mã | Ở đâu | Ý nghĩa |
|---|---|---|
| `CREATE-00` | Controller | API trả về cho người gọi |
| `CREATE-01..03` | `CreateCashbackCampaignServiceV2` | Sinh id, lưu `INITIALIZING`, flush |
| `CREATE-04` | — | **Transaction tạo campaign commit** |
| `CREATE-05/06` | `startSagaFlowAsync` | Bàn giao create saga sang luồng khác |
| `CREATE-07/08` | Saga create | Gửi event khởi động + **kết quả** (`ACCEPTED`/`DENIED`) |
| `SAGA-EVT` | `SagaEventHelper` | Kết quả mọi event khác trong create saga |
| `ACT-01..04` | `SagaStateChangeListener` | Create saga xong → đổi sang `ACTIVE` → lập lịch |
| `ACT-02c` | — | **Commit của lần đổi trạng thái** |
| `SCHED-01/02` | `scheduleCampaign` | Ghi lịch kích hoạt |
| `SCHED-04..09` | `processAction` | Mở transaction, đọc campaign, ghi `ENABLING`, gọi orchestrator |
| `SCHED-09c` | — | **Commit của scheduler** ← mốc quyết định |
| `SCHED-10..12` | Orchestrator | Hoãn hay chạy ngay; saga thực sự khởi động |
| `ENABLE-01/02` | Action + Consumer | Gửi lệnh cashback / nhận phản hồi |
| `ENABLE-03/04` | `UpdateCampaignToRunningAction` | **Trạng thái đọc được** / kết quả cuối |

### Cách đọc kết quả

| Quan sát | Kết luận |
|---|---|
| `SCHED-09c` xuất hiện **trước** `ENABLE-03` | Dữ liệu đã hiện hữu — nếu vẫn hỏng thì **không phải** race condition |
| `SCHED-09c` xuất hiện **sau** `ENABLE-03` | Vẫn là race — bản vá chưa ăn |
| `CREATE-08 resultType=DENIED` | Saga create không khởi động được |
| `SAGA-EVT resultType=DENIED` | Chỉ ra chính xác bước nào trong chuỗi bị chặn |

Cách kiểm tra nhanh nhất, không cần đọc kỹ: xem thứ tự hai dòng

```
Campaign X ACTIVATE completed successfully     ← commit
Processing cashback enabled: campaignId=X      ← phản hồi
```

Nếu dòng dưới nhảy lên trên dòng trên, campaign sẽ kẹt.

---

## 6. Việc cần làm khi deploy

### 6.1 Build lại

Các thay đổi **chưa được biên dịch** trong môi trường phân tích. Cần chạy build trước khi deploy.

### 6.2 Dọn dữ liệu tồn (bắt buộc, chạy **sau** khi deploy)

Việc gỡ bỏ các state trung gian khiến những bản ghi đang nằm ở các state đó **không còn đường đi tiếp**. Nếu để lại, `SagaTimeoutMonitor` sẽ quét lại vô tận.

Cần xoá các bản ghi trong bảng `state_machine` có `state` thuộc:
- `START_CREATE`
- `CASHBACK_CAMPAIGN_ENABLE_PENDING`
- `CASHBACK_ENABLE_COMPENSATING`

Xoá là an toàn: khi không tìm thấy ngữ cảnh cũ, hệ thống khởi tạo saga mới từ đầu.

Kịch bản SQL tham khảo: `scripts/fix-stuck-enabling-campaigns.sql` (file này nằm ngoài git theo chính sách `.gitignore` của repo — chỉ `src/`, `Jenkinsfile`, `pom.xml` được track).

### 6.3 Xử lý campaign đang kẹt

| Campaign | Trạng thái | Xử lý đề xuất |
|---|---|---|
| `019ffe84-37b0-724d-a9ac-725f4e9fa3c4` | `INITIALIZING` | Chuyển `ERROR` rồi tạo lại |
| `019ffacf-5e8a-7ca9-921e-5be0f4995a81` | `ENABLING` | Chuyển `RUNNING` — log cho thấy cashback phía kia **đã bật thật** |
| Nhóm campaign ngày 13/08 | `ENABLING` | Trả về `ACTIVE` rồi để scheduler kích hoạt lại |

### 6.4 Lưu ý về hành vi sau khi deploy

Sau bản vá vấn đề 5, những campaign trước đây kẹt im lặng sẽ **chuyển sang `ERROR` sau ~30–50 giây**. Đây là hành vi đúng — lỗi cũ giờ hiện ra, không phải lỗi mới phát sinh.

---

## 7. Còn tồn đọng — chưa xử lý

Các vấn đề đã xác định nhưng **cố ý chưa sửa** để giữ phạm vi thay đổi nhỏ, dễ kiểm chứng:

| Vấn đề | Vị trí | Rủi ro |
|---|---|---|
| `triggerEnableCashbackSaga` nuốt exception | `CampaignActivationSchedulerService` | Saga không khởi động được thì campaign kẹt `ENABLING`, bản ghi lịch vẫn báo `COMPLETED`, không retry |
| `markAsCompleted` gọi sai thời điểm | `CampaignActivationSchedulerService` | Bảng lịch báo "hoàn tất" ngay khi saga *khởi động*, trong khi campaign mới ở `ENABLING` |
| Tự khoá chính mình (self-deadlock) | `processAction` + `handleActionFailure` | Giữ khoá bi quan trên bản ghi rồi gọi transaction mới lên đúng bản ghi đó — chỉ thoát khi hết `innodb_lock_wait_timeout` (mặc định 50s) |
| `releaseStateMachine` thao tác nhầm đối tượng | `EnableStateMachineServiceImpl` | Tạo instance mới thay vì dùng lại cái đã lấy → cảnh báo `Cannot persist null state`. Vô hại nhưng gây nhiễu log |
| `CashbackEnableAction` vẫn tự gửi event ở nhánh lỗi | `application/action/enable` | Có thể dính `DENIED` nếu lỗi xảy ra tức thì. Hậu quả giới hạn: timeout monitor vẫn huỷ và hoàn tác đúng sau ~30s |
| Chặn 30 giây trên luồng dùng chung | `CashbackEnableAction` | Chiếm một worker của pool `Schedulers.parallel()` (chỉ `max(số nhân, 4)`) mỗi lần bật cashback |

---

## 8. Bài học rút ra

1. **Không để một state action tự gửi event ngược vào state machine đang chạy nó.** Đây là gốc của cả vấn đề 1 và 4. Action chạy bất đồng bộ song song với transition, nên event luôn có nguy cơ tới quá sớm. Mọi event nên đến từ bên ngoài: người khởi động, consumer Kafka, hoặc bộ giám sát timeout.

2. **Không gửi thông điệp ra ngoài khi transaction còn mở.** Bên nhận có thể trả lời trước khi dữ liệu kịp hiện hữu. Đây là gốc của vấn đề 3.

3. **Race condition không lộ trên môi trường nhanh.** Cùng một đoạn mã, sandbox thắng 20–30ms, UAT thua 6–12ms. Test trên môi trường có độ trễ giống thật, hoặc kiểm chứng bằng thứ tự sự kiện thay vì bằng kết quả cuối.

4. **Trạng thái trung gian phải luôn có đường thoát.** Mỗi trạng thái "đang xử lý" cần có bộ đếm giờ và một lối ra dẫn tới `ERROR`. Không có lối thoát thì lỗi biến thành im lặng — và im lặng là thứ tốn nhiều thời gian điều tra nhất trong toàn bộ sự cố này.

5. **Ghi lại kết quả của thao tác, không chỉ ghi lại việc đã gọi.** `.subscribe()` trần và `log.debug` là hai thứ đã che giấu nguyên nhân suốt nhiều ngày.
