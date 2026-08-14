# Lộ trình học State Machine & Saga

**Mục tiêu:** hiểu đủ sâu để thiết kế saga không lặp lại các lỗi đã gặp ở `promotion-campaign`.

Toàn bộ bài tập xây dần **một hệ thống duy nhất — saga đặt vé xem phim**. Mỗi giai đoạn thêm một lớp; đến cuối bạn có một saga hoàn chỉnh, tự tay gặp và tự tay sửa đúng những lỗi đã làm campaign kẹt trạng thái.

Chọn miền "đặt vé" thay vì "campaign" là có chủ ý: đủ giống để chuyển được kinh nghiệm, đủ khác để không copy-paste được.

> Bài toán xuyên suốt: **Giữ ghế → Trừ tiền → Phát vé**. Hỏng ở bước nào thì phải hoàn tác các bước trước đó.

---

## Bảng đối chiếu: lỗi đã gặp ↔ giai đoạn học

Đọc bảng này trước. Nó là lý do lộ trình được sắp xếp như vậy.

| Lỗi thật đã gặp | Hậu quả | Giai đoạn |
|---|---|---|
| Action tự gửi event ngược vào máy đang chạy nó → `DENIED` | Campaign kẹt `INITIALIZING`/`ENABLING`/`DISABLING` | **2** |
| Không phân biệt entry / do / transition action | Không hiểu vì sao action chạy trên thread khác | **2** |
| Gửi lệnh Kafka khi transaction chưa commit | Bên nhận đọc dữ liệu cũ, saga hoàn tất nhưng vô hiệu | **5** |
| State trung gian không có đường thoát | Kẹt im lặng, không báo lỗi | **6** |
| `.subscribe()` trần, `log.debug` cho kết quả | Lỗi không để lại dấu vết, mất nhiều ngày điều tra | **6** |
| Compensation không khớp trạng thái thực tế | Huỷ saga xong campaign vẫn kẹt | **4** |
| Chỉ test trên DB local | Race không bao giờ lộ | **7** |

---

## Giai đoạn 0 — Máy trạng thái, không framework

**Thời lượng gợi ý:** 1 buổi

### Học gì

Năm khái niệm, không hơn:

| Khái niệm | Nghĩa |
|---|---|
| **State** | Máy đang ở đâu |
| **Event** | Thứ kích hoạt chuyển đổi |
| **Transition** | `state nguồn + event → state đích` |
| **Guard** | Điều kiện cho phép transition |
| **Action** | Việc làm kèm theo transition |

Điều quan trọng nhất cần thấm: **transition chỉ xảy ra khi có đúng cặp (state hiện tại, event)**. Nếu máy chưa ở đúng state, event bị từ chối. Toàn bộ sự cố `DENIED` của chúng ta chỉ là hệ quả của câu này.

### Bài tập

Viết máy trạng thái đặt vé bằng **enum + switch thuần Java**, không framework:

```
NEW → SEAT_HELD → PAID → TICKET_ISSUED
                ↘ CANCELLED
```

Yêu cầu:
- Hàm `handle(Event e)` trả về `ACCEPTED` hoặc `DENIED`
- Gửi `PAY` khi đang ở `NEW` phải trả `DENIED`
- Viết unit test cho **cả hai** trường hợp, đặc biệt là ca `DENIED`

### Tự kiểm tra

Không nhìn code, trả lời: *máy đang ở `NEW`, gửi event `ISSUE_TICKET` thì chuyện gì xảy ra, và ai chịu trách nhiệm phát hiện điều đó?*

---

## Giai đoạn 1 — Spring Statemachine, phần khung

**Thời lượng gợi ý:** 1–2 buổi

### Học gì

- `@EnableStateMachineFactory`, `EnumStateMachineConfigurerAdapter`
- `configure(StateMachineStateConfigurer)` — khai báo state, `initial`, `end`
- `configure(StateMachineTransitionConfigurer)` — `withExternal().source().target().event()`
- `sendEvent(Mono<Message>)` trả về `Flux<StateMachineEventResult>`
- **Ba giá trị `ResultType`: `ACCEPTED`, `DENIED`, `DEFERRED`**

### Cạm bẫy #1 — Bỏ qua kết quả trả về

`sendEvent` trả về một `Flux`. Nếu bạn không đọc kết quả, một event bị từ chối sẽ **biến mất không dấu vết**.

```java
// SAI — DENIED biến mất, saga đứng im, log trống trơn
stateMachine.sendEvent(Mono.just(msg)).subscribe();

// ĐÚNG — luôn ghi lại resultType
stateMachine.sendEvent(Mono.just(msg))
        .subscribe(result -> log.info("event={} resultType={}", event, result.getResultType()),
                   error  -> log.error("event={} failed", event, error));
```

Đây chính xác là dòng code đã che giấu lỗi của saga create suốt nhiều ngày.

### Bài tập

Chuyển máy trạng thái ở giai đoạn 0 sang Spring Statemachine. Viết test khẳng định:
- Gửi `HOLD_SEAT` khi ở `NEW` → `ACCEPTED`
- Gửi `PAY` khi ở `NEW` → `DENIED`

Test thứ hai quan trọng hơn test thứ nhất.

### Tự kiểm tra

`DEFERRED` khác `DENIED` chỗ nào? Khi nào Spring Statemachine trả `DEFERRED`?

---

## Giai đoạn 2 — Action: nơi mọi lỗi bắt đầu

**Thời lượng gợi ý:** 2–3 buổi. **Đây là giai đoạn quan trọng nhất của cả lộ trình.**

### Học gì

Bốn chỗ có thể gắn action, và chúng **không giống nhau**:

| Loại | Khai báo | Chạy khi nào | Thread |
|---|---|---|---|
| **Transition action** | `.withExternal()...action(a)` | Trong lúc transition | Thread đang xử lý event |
| **Entry action** | `state(S, entryActions, exitActions)` | Khi vào state | Thread đang xử lý event |
| **Exit action** | như trên | Khi rời state | Thread đang xử lý event |
| **Do action** | `state(S, action)` ← **một tham số** | Sau khi vào state, **bất đồng bộ** | Scheduler riêng (`parallel-N`) |

Điểm chết người: `state(S, action)` **một tham số** là **do action**, không phải entry action. Nó chạy trên thread khác, **song song** với chính transition vừa đưa máy vào state đó.

Trong dự án chúng ta, comment ghi *"Actions are defined in STATE ENTRY"* — sai. Tất cả đều là do action.

### Cạm bẫy #2 — Action tự gửi event ngược vào máy đang chạy nó

Đây là lỗi đã làm campaign kẹt ở **cả ba** luồng create, enable, disable.

```java
// SAI — do action tự đẩy saga đi tiếp
public void execute(StateContext<S, E> ctx) {
    ctx.getStateMachine()
       .sendEvent(Mono.just(MessageBuilder.withPayload(NEXT_EVENT).build()))
       .subscribe();     // → DENIED, và không ai biết
}
```

Vì sao hỏng: do action chạy song song với transition. Khi nó gửi event, transition trước **chưa xong**, máy chưa sẵn sàng nhận event mới → `DENIED`. Và Spring Statemachine **không xếp lại hàng đợi** event đã bị từ chối — saga đứng im vĩnh viễn.

Vì sao nó "chạy được" trên máy dev: transition trên DB local xong trong vài mili giây, action gửi event sau đó nên vẫn kịp. Trên môi trường thật, transition mất hàng trăm mili giây vì phải ghi DB qua mạng. Số đo thật:

| Luồng | Event bị đánh giá | Transition hoàn tất | Biên |
|---|---:|---:|---:|
| create | +113ms | +122ms | −9ms |
| enable | +108ms | +258ms | −150ms |
| disable | +110ms | +135ms | −25ms |

**Quy tắc rút ra:** mọi event phải đến **từ bên ngoài** máy trạng thái — từ người khởi động, từ consumer nhận phản hồi, từ bộ giám sát timeout. Không state nào được tự đẩy mình đi.

### Cạm bẫy #3 — State pass-through

Nếu một state chỉ tồn tại để chạy một action rồi bắn event đi tiếp, **state đó không cần tồn tại**. Xoá nó và nối thẳng transition. Ba state như vậy đã bị xoá khỏi dự án.

```
Trước:  START → PASS_THROUGH → LÀM_VIỆC_THẬT
Sau:    START → LÀM_VIỆC_THẬT
```

### Cạm bẫy #4 — Spring Statemachine nuốt exception của action

Action ném exception thì saga chết im lặng. Luôn bọc try/catch và ghi log, hoặc override `stateMachineError` trên listener.

### Bài tập

Ba phần, làm đúng thứ tự:

1. **Tái tạo lỗi.** Thêm một state `VALIDATING` chỉ để chạy một do action tự gửi event đi tiếp. Chèn `Thread.sleep(200)` vào listener persistence để giả lập DB chậm. Quan sát `DENIED`.
2. **Sửa bằng cách xoá state.** Nối thẳng transition. Xác nhận hết `DENIED`.
3. **So sánh thread.** In `Thread.currentThread().getName()` trong cả bốn loại action. Tự mắt thấy do action chạy trên thread khác.

Phần 1 là phần giá trị nhất. Đừng bỏ qua.

### Tự kiểm tra

Nhìn dòng `.state(S.PAID, chargeAction)` — action này chạy trên thread nào, và nếu nó gọi `sendEvent` thì có an toàn không?

---

## Giai đoạn 3 — Lưu trạng thái và khôi phục

**Thời lượng gợi ý:** 1–2 buổi

### Học gì

- `StateMachinePersist` / `StateMachineRuntimePersister`
- `machineId` — khoá định danh một phiên saga
- `acquireStateMachine` / `releaseStateMachine`
- Extended state (`getExtendedState().getVariables()`) — nơi mang dữ liệu nghiệp vụ qua các bước

### Cạm bẫy #5 — Đặt tên machineId trùng nhau

Một campaign có thể có nhiều saga (create, enable, disable). Nếu cả ba dùng chung `machineId = campaignId`, chúng ghi đè nhau. Dự án giải quyết bằng tiền tố: `enable:<id>`, `disable:<id>`.

### Cạm bẫy #6 — Đổi hình dạng máy khi dữ liệu cũ còn nằm trong DB

Khi bạn xoá một state, các bản ghi đang ở state đó **không còn đường đi tiếp**. Chúng sẽ kẹt vĩnh viễn và làm bộ giám sát timeout quét lại vô tận. Mỗi lần đổi cấu trúc máy phải có kế hoạch dọn dữ liệu đi kèm — dự án phải xoá 6 state khỏi bảng `state_machine`.

### Bài tập

- Thêm persistence (JPA hoặc in-memory) cho saga đặt vé
- Đặt `machineId = "booking:" + bookingId`
- Dừng ứng dụng giữa lúc saga đang ở `SEAT_HELD`, khởi động lại, gửi event tiếp theo, xác nhận saga chạy tiếp đúng
- Thử xoá một state khỏi config trong khi DB còn bản ghi ở state đó — quan sát hậu quả

### Tự kiểm tra

Extended state được lưu ở đâu, và điều gì xảy ra với nó khi máy được khôi phục?

---

## Giai đoạn 4 — Saga: điều phối nhiều service

**Thời lượng gợi ý:** 2–3 buổi

### Học gì

- **Orchestration** (một bộ điều phối ra lệnh) vs **choreography** (mỗi service tự phản ứng). Dự án dùng orchestration.
- Vòng đời một bước: gửi **command** → chờ **event phản hồi** → chuyển state
- **Correlation** — ghép phản hồi về đúng saga đang chờ
- **Compensation** — hoàn tác các bước đã thành công khi bước sau hỏng

### Cạm bẫy #7 — Compensation không khớp trạng thái thực tế

Ở dự án, hành động hoàn tác chỉ chạy khi campaign ở `RUNNING`, nhưng luồng scheduler để campaign ở `ENABLING`. Nó ghi một dòng cảnh báo rồi bỏ qua — campaign kẹt.

Bài học: compensation phải xử lý **mọi trạng thái mà nó thực sự có thể gặp**, không chỉ trạng thái bạn hình dung khi viết. Và phải khôi phục về **trạng thái trước đó thật sự**, nên hãy lưu `previousStatus` vào extended state ngay từ đầu.

### Cạm bẫy #8 — Chuỗi compensation dài không cần thiết

Luồng disable có hai state compensation nối nhau, và hoá ra **hai action làm y hệt nhau**. Mỗi mắt xích thừa là một chỗ nữa để hỏng. Compensation nên là transition action gắn thẳng vào đường huỷ.

### Bài tập

Tách saga đặt vé thành ba service thật (hoặc ba module gọi qua queue nội bộ):
- `seat-service` — giữ / nhả ghế
- `payment-service` — trừ tiền / hoàn tiền
- `ticket-service` — phát vé

Yêu cầu:
- Bộ điều phối gửi command, chờ event, không gọi trực tiếp
- Ép `payment-service` fail → xác nhận ghế được nhả
- Lưu `previousStatus` và dùng nó khi hoàn tác

### Tự kiểm tra

Nếu `payment-service` trả lời **hai lần** cho cùng một command, saga có chạy sai không?

---

## Giai đoạn 5 — Transaction và tính nhìn thấy được

**Thời lượng gợi ý:** 2 buổi. **Giai đoạn dễ bị xem nhẹ nhất, và tốn nhiều ngày điều tra nhất.**

### Học gì

- Ranh giới transaction của Spring (`@Transactional`, `REQUIRES_NEW`)
- Dữ liệu đã ghi **chưa nhìn thấy được** từ thread/transaction khác cho tới khi commit
- `TransactionSynchronization.afterCommit`
- Mức cô lập: MariaDB mặc định `REPEATABLE READ` — transaction đọc chốt ảnh chụp ngay lúc **mở**, không phải lúc đọc

### Cạm bẫy #9 — Gửi thông điệp ra ngoài khi transaction còn mở

Lỗi tốn nhiều thời gian nhất của cả sự cố:

```java
// SAI
@Transactional
public void enable(String id) {
    campaign.setStatus(ENABLING);
    campaignPort.save(campaign);        // chưa commit
    orchestrator.startSaga(id, ctx);    // đã gửi lệnh Kafka đi rồi
}
```

Bên nhận trả lời sau 92ms. Transaction commit sau 331ms. Người xử lý phản hồi đọc DB, thấy trạng thái **cũ**, không làm gì, và campaign kẹt vĩnh viễn.

```java
// ĐÚNG — hoãn tới sau commit
if (TransactionSynchronizationManager.isActualTransactionActive()) {
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override public void afterCommit() { orchestrator.startSaga(id, ctx); }
    });
} else {
    orchestrator.startSaga(id, ctx);
}
```

Giải pháp bài bản hơn cho hệ thống lớn: **transactional outbox** — ghi thông điệp vào cùng transaction với dữ liệu, một tiến trình riêng đọc bảng outbox và phát đi. Nên đọc để biết, kể cả khi chưa dùng.

### Cạm bẫy #10 — Tự khoá chính mình

Giữ khoá bi quan trên một bản ghi rồi mở transaction mới (`REQUIRES_NEW`) ghi lên đúng bản ghi đó → transaction con chờ transaction cha, cha chờ con. Chỉ thoát khi hết `innodb_lock_wait_timeout` (mặc định 50 giây).

### Bài tập

1. Ghi trạng thái `SEAT_HELD` trong transaction rồi gửi command ngay lập tức
2. Cho service nhận trả lời sau đúng 50ms, transaction giữ mở 300ms (`Thread.sleep`)
3. Quan sát bên nhận đọc ra trạng thái cũ — **tự tay tái tạo lỗi**
4. Sửa bằng `afterCommit`, xác nhận hết
5. Đọc thêm về transactional outbox và viết một đoạn ngắn so sánh hai cách

### Tự kiểm tra

Vì sao lỗi này chỉ xuất hiện trên môi trường có DB ở xa? Con số nào quyết định thắng thua?

---

## Giai đoạn 6 — Timeout, idempotency, khả năng quan sát

**Thời lượng gợi ý:** 2 buổi

### Học gì

- Mỗi state trung gian phải có **bộ đếm giờ và một lối thoát**
- Consumer phải **idempotent** — Kafka giao ít nhất một lần
- Ghi log **kết quả**, không chỉ ghi việc đã gọi

### Cạm bẫy #11 — State trung gian không có đường thoát

Nếu một state chỉ có transition khi mọi thứ thuận lợi, thì khi có sự cố nó là ngõ cụt. Mỗi state "đang chờ" cần:
- Một transition thành công
- Một transition thất bại
- Một bộ giám sát bắn transition thất bại khi quá hạn

### Cạm bẫy #12 — Huỷ saga mà quên cập nhật thực thể nghiệp vụ

Bộ giám sát timeout của dự án đánh dấu saga `ABORTED` trong bảng nội bộ nhưng **không đổi trạng thái campaign**. Người dùng thấy "đang xử lý" mãi mãi, không có gì để retry.

**Huỷ saga và cập nhật thực thể phải luôn đi cùng nhau.**

### Cạm bẫy #13 — Log ở mức không ai bật

`log.debug("resultType={}")` vô dụng trên môi trường chạy ở mức INFO. Thứ quan trọng phải ghi ở mức đủ để nhìn thấy.

Nên có sẵn một tiện ích trace ngay từ đầu — có tiền tố định danh, tên thread, và **trạng thái transaction**:

```
[<bookingId>] <BƯỚC> | thread=<...> | tx=ACTIVE(...) | <nội dung>
```

Riêng trường `tx=` đã trả lời được câu hỏi "dữ liệu này đã nhìn thấy được chưa" mà không cần suy đoán.

### Bài tập

- Thêm bộ giám sát timeout: state `PAYMENT_PENDING` quá 30 giây → bắn event thất bại
- Đảm bảo khi saga huỷ thì booking chuyển sang `FAILED`, không nằm lại `PAYMENT_PENDING`
- Gửi cùng một event phản hồi hai lần, xác nhận không phát hai vé
- Viết tiện ích trace log như trên và dùng xuyên suốt

### Tự kiểm tra

Liệt kê mọi state trung gian trong saga của bạn. State nào chưa có đường thoát khi sự cố?

---

## Giai đoạn 7 — Kiểm thử để lỗi lộ ra trước khi lên môi trường thật

**Thời lượng gợi ý:** 2 buổi

### Học gì

- `StateMachineTestPlanBuilder` của Spring Statemachine
- Testcontainers — chạy test với DB thật thay vì H2
- Chèn độ trễ nhân tạo để ép race lộ ra

### Cạm bẫy #14 — Test trên môi trường quá nhanh

Toàn bộ sự cố không lộ ở dev vì DB local nhanh gấp 9 lần. Biên thắng ở dev chỉ **20–30ms** — không phải "đúng", chỉ là "chưa lộ".

Hai cách chống:
1. **Chèn độ trễ:** thêm `Thread.sleep` vào listener persistence trong test, ép transaction dài ra
2. **Khẳng định thứ tự, không khẳng định kết quả:** viết test kiểm tra *commit xảy ra trước khi command được gửi*, thay vì chỉ kiểm tra trạng thái cuối

Cách 2 mạnh hơn vì nó không phụ thuộc tốc độ máy.

### Bài tập

- Viết test khẳng định: với mọi event, `resultType` phải là `ACCEPTED` — không bao giờ `DENIED`
- Viết test chèn độ trễ 300ms vào transaction, xác nhận saga vẫn đúng
- Chạy toàn bộ test với Testcontainers MariaDB thay vì H2, so sánh thời gian và kết quả

### Tự kiểm tra

Nếu ngày mai DB chậm đi gấp đôi, test nào của bạn sẽ đỏ? Nếu không có test nào đỏ, bạn chưa test đúng thứ cần test.

---

## Bảy quy tắc rút từ sự cố thật

Dán lên bàn làm việc. Mỗi quy tắc tương ứng với ít nhất một ngày điều tra.

1. **Không state nào tự đẩy mình đi.** Mọi event đến từ bên ngoài máy trạng thái.
2. **Không gửi thông điệp ra ngoài khi transaction còn mở.** Commit trước, gửi sau.
3. **Mỗi state trung gian phải có bộ đếm giờ và lối thoát dẫn tới trạng thái lỗi.**
4. **Huỷ saga thì phải cập nhật thực thể nghiệp vụ.** Hai việc này không tách rời.
5. **Luôn ghi lại `resultType`.** Một event bị từ chối mà không ai ghi lại là một sự cố im lặng.
6. **State chỉ để chuyển tiếp thì không nên tồn tại.**
7. **Race không lộ trên máy nhanh.** Kiểm chứng bằng thứ tự sự kiện, không bằng kết quả cuối.

---

## Danh sách kiểm tra khi review code saga

Dùng khi review PR có đụng tới state machine:

- [ ] Có action nào gọi `sendEvent` lên chính máy đang chạy nó không?
- [ ] `state(S, action)` một tham số — người viết có biết đó là **do action** chạy async không?
- [ ] Mọi `sendEvent` có ghi lại `resultType` ở mức INFO trở lên không?
- [ ] Có state nào chỉ tồn tại để chuyển tiếp không?
- [ ] Command/event có được gửi bên trong `@Transactional` không?
- [ ] Mỗi state "đang chờ" có đường thoát khi timeout không?
- [ ] Đường huỷ saga có cập nhật trạng thái thực thể nghiệp vụ không?
- [ ] Compensation có xử lý **mọi** trạng thái thực tế có thể gặp không?
- [ ] Có lưu trạng thái trước đó để khôi phục đúng không?
- [ ] Consumer có idempotent không?
- [ ] Nếu đổi hình dạng máy — đã có kế hoạch dọn dữ liệu cũ chưa?
- [ ] Action có bọc try/catch và ghi log không? (framework nuốt exception)

---

## Tài liệu nên đọc

| Chủ đề | Nguồn |
|---|---|
| Spring Statemachine | Tài liệu chính thức, đọc kỹ chương *State Actions* và *Persisting* |
| Saga pattern | Chris Richardson — *Microservices Patterns*, chương 4 |
| Transactional outbox | microservices.io — mẫu *Transactional Outbox* |
| Mức cô lập giao dịch | Tài liệu MariaDB về `REPEATABLE READ` và MVCC |
| Kiểm thử hệ bất đồng bộ | Testcontainers, và *Awaitility* cho khẳng định có chờ |

---

## Cách tự đánh giá đã học xong

Bạn hoàn thành lộ trình khi làm được ba việc sau **mà không cần tra cứu**:

1. Nhìn một đoạn config state machine, chỉ ra ngay action nào chạy đồng bộ, action nào chạy async.
2. Nhìn một service `@Transactional` có gửi message, nói được chỗ nào sẽ hỏng khi DB chậm.
3. Cầm log của một saga bị kẹt, xác định được nó kẹt ở đâu và vì sao — trong vòng 5 phút.

Việc thứ ba là thước đo thật. Sự cố ở `promotion-campaign` mất nhiều ngày chủ yếu vì log không đủ để trả lời câu hỏi đó.
