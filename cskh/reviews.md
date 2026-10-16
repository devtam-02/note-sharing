# Review PR #96 — promotion-vtm-bff (từng commit)

- **PR:** http://bitbucket.digital.vn/projects/PROMOTION/repos/promotion-vtm-bff/pull-requests/96/overview
- **Tiêu đề:** Tamntt25/cskh customer care api
- **Nhánh:** `tamntt25/cskh-customer-care-api` → `staging` (OPEN). Tác giả: tamntt25
- **Quy mô:** 13 commit, 129 file, +6832 / −20
- **Ngày review:** 2026-10-15. Đầu nhánh lúc review: `5685a0c25f`
- **Tiêu chí:** đúng/sai logic, idempotency/race, transaction, migration, hiệu năng, bảo mật (authz, PII), contract API, test, Sonar S3776/S1192/S125/S1128

## Kết luận chung: ⚠️ CẦN SỬA TRƯỚC KHI MERGE (không còn blocker)

Không còn finding 🔴 nào ở đầu nhánh. Tuy vậy PR còn nhiều lỗi 🟠 về đúng sai dữ liệu: gán nhầm khách với mã chung, mất event lịch sử, backfill lặp vô hạn, API chi tiết chiến dịch dễ lỗi 500 khi tải cao. Nên sửa các lỗi này trước khi merge vào staging.

**Phạm vi đã loại (theo xác nhận của chủ PR, 2026-10-16):**
- Xác thực/phân quyền cho vùng `/api/cskh/v1/**`: được triển khai theo hướng khác, ngoài PR này. Các finding liên quan vẫn giữ lại trong file với trạng thái `⚪ Dismissed`, không tính vào số đếm.
- `InternalBackfillController` (`/internal/backfill`): chỉ dùng nội bộ, gateway chỉ mở các path `/api/v1/` ra ngoài. Không phải lỗ hổng.

### Bảng tổng hợp

| # | Commit | Nội dung | Đánh giá | 🔴 | 🟠 | 🟡 | 🔵 |
|---|---|---|---|---|---|---|---|
| 1 | 71f689e | API spec | ⚠️ | 0 | 1 | 4 | 2 |
| 2 | 41e1902 | Read model schema | ⚠️ | 0 | 1 | 2 | 2 |
| 3 | 9f26958 | Backfill campaign_code/coupon_type | ❌ tại commit (đã sửa ở 4c45595) | 2 | 1 | 3 | 2 |
| 4 | 0a20898 | Fill campaign_code/coupon_type live path | ⚠️ | 0 | 0 | 3 | 2 |
| 5 | 4c45595 | Assignment status + campaign stock | ⚠️ | 0 | 2 | 7 | 2 |
| 6 | b30d84d | Backfill assignment status | ⚠️ | 0 | 1 | 3 | 1 |
| 7 | 1ff368a | Project offer history/usage | ⚠️ | 0 | 6 | 4 | 2 |
| 8 | 489122a | Customer offer search | ⚠️ | 0 | 3 | 6 | 0 |
| 9 | d2df587 | Offers khách chưa có code | ⚠️ | 0 | 2 | 2 | 0 |
| 10 | fc62ac2 | Msisdn 0xxx / 84xxx | ✅ | 0 | 0 | 1 | 2 |
| 11 | 90cc652 | Offer audit trail | ⚠️ (nhẹ) | 0 | 0 | 6 | 0 |
| 12 | cc01d14 | Fix applicableProducts | ⚠️ | 0 | 1 | 2 | 4 |
| 13 | 5685a0c | API gộp chi tiết chiến dịch | ⚠️ | 0 | 3 | 9 | 4 |

> Số đếm đã loại 4 finding về xác thực `/api/cskh/v1` (ở 71f689e, 489122a, 90cc652, 5685a0c). 🔴 duy nhất còn lại ở 9f26958 (map trùng cột `campaign_type` khiến app không khởi động, test không compile) đã sửa ở 4c45595. Hệ quả là commit đó không bisect được.

### Việc cần làm trước khi merge (ưu tiên)

1. 🟠 **Gán nhầm khách với mã chung** (1ff368a, `VoucherOwnershipStoreAdapter` 347-353): lọc theo `customerId` không ra thì code lùi về *mọi* người giữ mã, nên một lần đổi mã ghi REDEEM cho tất cả khách.
2. 🟠 **Chống trùng `offer_audit`**: khoá thiếu msisdn/customer (mã chung chỉ ghi được cho khách đầu tiên). Việc chống trùng là check-then-act, không có unique constraint. Cần changeset mới thêm unique key.
3. 🟠 **Consumer nuốt lỗi** (1ff368a): bắt `RuntimeException`, log rồi vẫn ack, làm mất event của lịch sử append-only. Cần retry/DLT.
4. 🟠 **Backfill không dừng**: b30d84d `AND status <> 'REVOKED'` bỏ sót dòng `status` NULL (cột nullable, changeset 005), nên `hasNext` luôn true. 9f26958 cũng lặp mãi `page=0` khi còn row không lấp được. Nên phân trang keyset và tính `hasNext` theo số dòng thực ghi.
5. 🟠 **Executor `campaignDetailExecutor`** (5685a0c, `BffAsyncConfig` 55-65): queue 50, AbortPolicy mặc định. Khoảng 9-10 request đồng thời với downstream chậm là `RejectedExecutionException`, thành HTTP 500. `join()` không có timeout, không có circuit breaker, Feign read-timeout 60s.
6. 🟠 **Search sai so với spec** (489122a/d2df587): lọc `productCode` bằng `LIKE '%X%'` trên JSON (khớp cả sản phẩm loại trừ và mã con), sort lệch spec. Nhánh bản ghi giả lập không lọc segment (số bất kỳ cũng thấy mọi chiến dịch). INNER JOIN `campaign_stock` làm rơi chiến dịch hoàn tiền.
7. 🟠 Giá trị ngoài contract: `assignment_status='revoked'`. `campaign_stock` cũ do không refresh khi phát/thu hồi mã. `SourceAvailability` báo sai "nguồn lỗi" khi chiến dịch chỉ là không khai phần đó. `applicableProducts=[]` bị hiểu là "mọi sản phẩm" khi JSON hỏng hoặc chỉ có loại trừ.
8. 🟠 Thiếu index `voucher_ownership.voucher_code` (mỗi event đổi mã full scan). Log `warn` in nguyên event có `customerPhone` (cần xác nhận `toString`).
9. 🟡 Test: adapter SQL, consumer và web layer (400/404/502) chưa có test. Một số test chỉ chạy với store mock nên che lỗi (vd `shared_code_writes_one_row_per_owner`).

### Ghi chú chung
- Không phát hiện SQL injection: mọi tham số đều bind, ORDER BY cố định. Tiền dùng `BigDecimal`, thời gian dùng `Instant`. Migration là file mới và immutable.
- Sonar: chỉ có 1 vi phạm S1128 (import trùng `CurrentTraceContext`, 5685a0c). Không thấy S125/S1192/S3776.
- Header `// id: … | dd_version:` không có ở file Java nào trong repo (0/272), nên không tính là finding riêng của PR.
- Changeset 024 để `author="claude"`.

---

# Chi tiết từng commit

## 71f689e — feat(cskh): add API spec for customer care offer lookup
**Phạm vi:** Thêm interface đặc tả `CustomerCareApi` (3 endpoint dưới `/api/cskh/v1`), `CustomerCareController` (api-specs chạy thật, search/audit trả 501) và 4 DTO record. 6 file, +602 dòng.
**Đánh giá:** ⚠️ Cần sửa

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| ⚪ Dismissed (ngoài phạm vi — auth CSKH triển khai theo hướng khác) | `CustomerCareController.java:46-52`, `CustomerCareApi.java:36-41` | Vùng `/api/cskh/v1` không có filter xác thực nào (`VtmAuthFilter` chỉ lọc `/api/v1/`). Ở commit này hai endpoint còn trả 501 nên chưa lộ dữ liệu, nhưng ở đầu nhánh (5685a0c) search/audit/campaign-detail đã trả dữ liệu thật mà vẫn chưa có filter (javadoc "Cảnh báo bảo mật" còn nguyên, không có filter/SecurityFilterChain nào cho `cskh`). Kết quả: ai gọi được tới service là tra được ưu đãi + lịch sử của bất kỳ msisdn nào (PII), dù spec ghi 401/403 | Bổ sung filter xác thực nhân viên/service-to-service (Keycloak client-credentials hoặc mTLS) và kiểm tra role CSKH TRƯỚC khi merge PR. Nếu chưa kịp thì chặn ở gateway/network policy và ghi rõ trong PR |
| 🟠 Major | `CustomerCareController.java:64` | `OpenApiWebMvcResource` là dependency bắt buộc (field `final`), nhưng bean này chỉ được springdoc tạo khi `springdoc.api-docs.enabled=true`. Môi trường nào tắt api-docs (hay gặp ở prod) thì cả context không khởi động được, chứ không trả 503 như spec ghi ở `CustomerCareApi.java:76` | Inject `ObjectProvider<OpenApiWebMvcResource>` rồi trả 503 khi không có bean. Xác nhận config prod/staging bên ngoài repo có bật api-docs không |
| 🟡 Minor | `CustomerCareController.java:142` | `(ObjectNode) full.path("components").path("schemas")`: khi tài liệu không có `components.schemas`, `path()` trả `MissingNode` nên ép kiểu ném `ClassCastException` (500) trước khi tới dòng kiểm tra `isMissingNode()` ở dòng 143. Lỗi vẫn còn ở 5685a0c (dòng 160) | Đọc ra `JsonNode` trước, kiểm tra `isObject()` rồi mới ép kiểu |
| 🟡 Minor | `CustomerCareController.java:84-87` | `catch (Exception e)` rồi đưa `e.getMessage()` vào lý do của `ResponseStatusException`, có thể lộ chi tiết nội bộ ra client (rule BE §8: không trả message exception thô) | Trả message cố định, chỉ log chi tiết; bắt exception cụ thể (`IOException`, `ClassCastException`) |
| 🟡 Minor | `CustomerCareApi.java:121,136,152,162` | Các tham số enum (`assignmentStatus`, `campaignStatus`, `usageStatus`, `offerType`) nhận String tự do, chỉ có `@Size(max=32)`. `allowableValues` chỉ là tài liệu. Ở 5685a0c giá trị sai (vd `foo`) được đưa thẳng vào WHERE nên trả 200 rỗng thay vì 400 như spec mô tả (vi phạm rule whitelist enum §4.4) | Parse về enum Java (hoặc thêm `@Pattern` theo danh sách) để trả 400 |
| 🟡 Minor | `CustomerCareApi.java:164` | Mô tả `productCode` nói "lấy từ API filter-options" nhưng không có API nào tên đó (dropdown lấy từ `/api-specs`). Vẫn còn ở 5685a0c | Sửa mô tả cho đúng nguồn danh mục sản phẩm |
| 🔵 Info | `CustomerOfferItem.java:27`, `AuditCampaignSummary.java:23` | Ví dụ `campaignStatus` là `RUNNING` (chữ hoa) trong khi giá trị lọc `campaignStatus` của search là chữ thường (`running`), dễ khiến bên tích hợp hiểu sai | Thống nhất một dạng giá trị, hoặc ghi rõ response trả nguyên enum phía nguồn |
| 🔵 Info | toàn commit | Không có test cho `getApiSpecs`/`trimToSearchOperation` (thuật toán gom `$ref` đệ quy). Các commit sau cũng không bổ sung (chỉ có `CustomerCareApiMergeTest` cho endpoint gộp) | Thêm unit test cho phần cắt gọn spec (có ref lồng nhau, không có schemas) |

**Ghi chú:** Thiết kế đặt tiền tố riêng để tránh token khách hàng lọt qua, và giải thích lý do `@ResponseWrapper` đặt ở mức method, đều hợp lý. Ràng buộc `to >= from` (spec trả 400) đã được cài ở 90cc652. Mọi query ở đầu nhánh đều parameterized, không có SQL injection qua các tham số enum.

## 41e1902 — feat(cskh): add read model schema for customer care offer lookup
**Phạm vi:** Liquibase 022 (thêm 6 cột vào `voucher_ownership`, `campaign_offer.campaign_code`, `coupon_display.coupon_type`, index (customer_source_id, campaign_id), tạo lại view `customer_voucher_view` từ 53 lên 61 cột) và 023 (bảng `offer_audit`, `campaign_stock`). 3 file, +247 dòng.
**Đánh giá:** ⚠️ Cần sửa

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| 🟠 Major | `023-cskh-audit-tables.xml:15-16,45-47,66-68` | `offer_audit` không có unique key nào để chống ghi trùng (`event_id` chỉ có index thường). Ở 5685a0c, consumer chống trùng bằng check-then-act `existsByEventIdAndVoucherCodeAndAction` (`OfferAuditStoreAdapter.java:28`), nên khi message bị giao lại song song hoặc consumer chạy nhiều instance thì vẫn ghi trùng (rule §2: chống trùng phải ở tầng DB) | Thêm cột `dedupe_key` NOT NULL (vd hash của event_id + voucher_code + action, vì `voucher_code` có thể NULL với Hoàn tiền) kèm unique index, rồi insert-first và bắt lỗi duplicate. Phải viết thành changeset MỚI, không sửa 023 |
| 🟡 Minor | `022-cskh-columns-and-view.xml:147-149` | Rollback của changeset tạo lại view chỉ `DROP VIEW` chứ không dựng lại định nghĩa của 020. Rollback 022 sẽ khiến app mất view mà luồng app khách hàng đang dùng (020 cũng làm vậy nhưng 020 là bản đầu) | Trong rollback, tạo lại view theo SELECT 53 cột của 020 |
| 🟡 Minor | `023-cskh-audit-tables.xml:63-65` | `idx_audit_voucher_code` chỉ có ích cho khớp bằng hoặc tiền tố. Spec lọc audit theo mã là "khớp tương đối" (`LIKE '%x%'`) nên index không được dùng. Trong thực tế query luôn có customer + campaign nên index tổ hợp đã đủ, index này chỉ làm chậm ghi | Bỏ index này, hoặc đổi spec sang khớp tiền tố/chính xác |
| 🔵 Info | `023-cskh-audit-tables.xml:18` | Comment ghi "Ba bang" nhưng changelog chỉ tạo 2 bảng | Sửa comment |
| 🔵 Info | `022…xml:27,46,55,64,74`, `023…xml:21,57,77` | `author="claude"`. Không sai về kỹ thuật, nhưng audit trail của migration nên ghi người chịu trách nhiệm | Đổi sang username dev (chỉ được đổi khi changeset chưa chạy ở môi trường nào, vì đổi author đổi luôn định danh changeset) |

**Ghi chú:** Đã đối chiếu: phần SELECT của view trùng khít với 020 (diff chỉ khác 8 cột thêm cuối). `context="dev"` đúng quy ước repo (`spring.liquibase.contexts=dev`). File 022/023 KHÔNG bị sửa lại ở các commit sau (sửa đổi schema về sau nằm ở 024 mới), nên giữ được tính immutable. Dùng `DATETIME(6)` + `Instant` là đúng rule thời gian. Index (customer_source_id, campaign_id) phục vụ đúng truy vấn tra cứu.

## 9f26958 — feat(cskh): backfill campaign_code and coupon_type
**Phạm vi:** Thêm 2 endpoint nội bộ `POST /internal/backfill/campaign-code` và `/coupon-type`, service backfill tra REST pp-campaign/pp-coupon theo trang, thêm field `code`/`couponType` vào DTO, snapshot và entity. 15 file, +246/-4.
**Đánh giá:** ❌ Chặn merge (ở trạng thái commit này; lỗi chặn đã được sửa ở 4c45595)

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| 🔴 Blocker | `CampaignOfferEntity.java:33-37` | Field mới chèn vào GIỮA `@Column(name = "campaign_type", length = 32)` và `private String campaignType`, nên annotation này gắn vào `campaignCode`. `campaignType` không còn annotation, và với `CamelCaseToUnderscoresNamingStrategy` cũng map vào `campaign_type`. Hibernate báo cột `campaign_type` bị map trùng, app không khởi động được. Kể cả khi khởi động được thì `campaign_code` cũng không bao giờ được ghi. **Đã sửa ở 4c45595** (thêm `@Column(name = "campaign_code", length = 64)` và trả annotation về đúng `campaignType`) | Commit này không bisect được. Nếu được thì squash phần sửa vào commit này |
| 🔴 Blocker | `CampaignDetailDto.java:23`, `CampaignDetailLookupPort.java:73` (test: `CampaignDetailLookupAdapterTest.java:85`, `CampaignValidityBackfillServiceTest.java:49`, `VoucherExpiryBackfillServiceTest.java:46`) | Đổi signature record (thêm `code`/`campaignCode`) nhưng không cập nhật 3 test đang gọi constructor với số tham số cũ, nên `mvn test` lỗi compile ở commit này. **Đã sửa ở 4c45595** | Như trên. Lần sau chạy test trước mỗi commit |
| 🟠 Major | `CskhCampaignFieldsBackfillService.java:36-41,58-61,73-79` | Row tra không ra (campaign/config đã xoá bên nguồn, hoặc nguồn không có code) được bỏ qua nhưng vẫn nằm trong tập "còn thiếu". Thế mà hướng dẫn vận hành (log ở dòng 76) lại bảo gọi lặp `page=0` cho tới khi `totalMissing=0`. Khi số row không bao giờ lấp được ≥ `size`, `page=0` trả mãi cùng một tập nên vòng lặp không kết thúc, và các row lấp được nằm sau bị bỏ sót. Job không resumable đúng nghĩa (rule §10) | Lặp theo keyset (`campaignId > :lastId`) thay vì theo offset trên tập đang co lại, hoặc đánh dấu row đã thử (cột/bảng trạng thái), và trả `lastId` trong `BackfillResult` |
| 🟡 Minor | `InternalBackfillController.java:181-199` | Một request đồng bộ có thể gọi REST tuần tự tới 500 lần (`@Max(500)`), mỗi lần kèm một transaction `findById`+`save` riêng. Dễ vượt timeout của gateway/LB, và client thấy lỗi dù server vẫn đang ghi | Giảm `@Max` (vd 100), hoặc chạy async rồi trả job id |
| 🟡 Minor | `CouponConfigLookupAdapter.java:43-59` | Chỉ bắt `FeignException`. Lỗi runtime nằm ngoài Feign (vd `CallNotPermittedException` của circuit breaker, hoặc lỗi mạng bị bọc bởi lớp khác) làm hỏng cả trang, trái với javadoc của port ("rỗng khi downstream lỗi, caller bỏ qua row") | Bắt thêm `RuntimeException`, log rồi trả `Optional.empty()` (cùng mẫu với `CampaignDetailLookupAdapter`) |
| 🟡 Minor | `CouponDisplayEntity.java:40-41` | `couponType` không có `@Column(name = "coupon_type", length = 32)` như các field khác. Hiện vẫn chạy nhờ naming strategy, nhưng không nhất quán và mất ràng buộc độ dài | Khai báo `@Column` tường minh |
| 🔵 Info | `CskhCampaignFieldsBackfillService.java` | Không có unit test cho service backfill mới (các commit sau cũng không thêm `CskhCampaignFieldsBackfillServiceTest`) | Thêm test: dryRun không ghi, row skip, đếm updated/skipped |
| 🔵 Info | `InternalBackfillController.java:18` | Import `CskhCampaignFieldsBackfillUseCase` chen giữa hai import `CouponDisplay*`, sai thứ tự alphabet (không phải lỗi Sonar) | Organize imports |

**Ghi chú:** Hướng tiếp cận (tra REST ngoài transaction, chỉ bọc transaction ngắn ở từng lần update trong adapter, `dryRun=true` mặc định) đúng rule "không gọi API ngoài trong @Transactional". Ghi lại cùng giá trị nhiều lần là vô hại nên backfill idempotent về giá trị. Endpoint `/internal/backfill` chỉ dùng nội bộ (đã xác nhận: gateway chỉ mở `/api/v1/` ra ngoài). Không có code bị comment-out, import thừa hay literal lặp ≥3 lần (S125/S1128/S1192 sạch). Javadoc của field `campaignCode` đặt sau `@Column` (dòng 34) cũng đã được sửa thứ tự ở 4c45595.

---

## 0a20898 — feat(cskh): fill campaign_code and coupon_type on the live path

**Phạm vi:** Sau khi projection campaign/coupon-config event ghi xong, consumer gọi thêm `ensureCampaignCode` / `ensureCouponTypeForCampaign` để lấp `campaign_code` / `coupon_type` bằng REST lookup (pp-campaign, pp-coupon) nếu row còn thiếu. 7 file, +102 dòng.

**Đánh giá:** ⚠️ Cần sửa (nhẹ)

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| 🟡 Minor | `src/test/.../messaging/CampaignProjectionConsumerTest.java:16`, `CouponDisplayProjectionConsumerTest.java:17` (tại 0a20898) | Commit thêm field `final` vào 2 consumer (đổi constructor của `@RequiredArgsConstructor`) nhưng không sửa test vẫn gọi `new CampaignProjectionConsumer(projection)` → **test không compile tại commit này**, phá khả năng bisect. | Sửa test trong cùng commit. **Đã sửa ở 4c45595.** |
| 🟡 Minor | `B/adapter/out/persistence/CampaignOfferBackfillAdapter.java:84-89` (`updateCampaignCode`), `:110-115` (`updateCouponType`) | Live path giờ gọi kiểu đọc-sửa-ghi (`findById` → `setX` → `save`) trên `campaign_offer`/`coupon_display`, không có `@Version`/`@DynamicUpdate`. Hibernate UPDATE ghi toàn bộ cột, nên nếu listener khác (coupon-config topic ghi discount/product scope vào `campaign_offer` qua `CustomerVoucherProjectionService`) ghi cùng row trong khoảng đó thì giá trị của nó bị ghi đè bằng bản cũ (lost update). Hai sự kiện tạo campaign và tạo coupon config thường đến gần nhau nên rủi ro này có thật. | Dùng atomic update có điều kiện: `UPDATE campaign_offer SET campaign_code=:c WHERE campaign_id=:id AND (campaign_code IS NULL OR campaign_code='')` (tương tự với coupon_type). Cách này vừa tránh lost update, vừa idempotent, vừa bỏ được câu `isCampaignCodeMissing` đọc trước. |
| 🟡 Minor | `B/application/service/CskhCampaignFieldsBackfillService.java:72-101` | Không có unit test cho `ensureCampaignCode` / `ensureCouponTypeForCampaign` (các nhánh: row chưa có, đã có code, lookup rỗng, nhiều config thiếu type). Kiểm tra đến 5685a0c vẫn chưa có test nào. | Bổ sung test service + test consumer (lỗi ở bước lấp không chặn ack, không chặn projection). |
| 🔵 Info | `B/application/service/CskhCampaignFieldsBackfillService.java:74` | Chỉ lấp khi đang thiếu, nên nếu pp-campaign cho sửa mã chiến dịch thì `campaign_code` ở read model đứng im mãi. | Nếu mã sửa được thì nên so sánh rồi cập nhật, hoặc ghi rõ giả định "mã bất biến" trong Javadoc. |
| 🔵 Info | `B/adapter/in/messaging/CampaignProjectionConsumer.java:48-53`, `CouponDisplayProjectionConsumer.java:48-53` | Gọi REST đồng bộ ngay trong Kafka listener (read-timeout Feign mặc định 60s, các client này không có circuit breaker). Commit message nói tải ở mức admin nên chấp nhận được, nhưng xem thêm finding tương ứng ở 4c45595, nơi số lần gọi tăng lên. | — |

**Ghi chú:**
- Tách hai khối try là đúng: lỗi ở bước lấp không làm hỏng projection đã ghi xong, và `continue` khi projection lỗi tránh được việc gọi lookup vô ích.
- Có index `idx_coupon_display_campaign_id` (changeset 017), nên query `findCouponConfigIdsWithoutCouponTypeByCampaign` không full scan.
- Tại commit này entity `CampaignOfferEntity` vẫn còn lỗi map `campaignCode` vào cột `campaign_type` (lỗi gốc từ 41e1902). Vì vậy `isCampaignCodeMissing`/`updateCampaignCode` thực tế đọc/ghi sai cột. **Đã sửa ở 4c45595.**
- Không có import thừa, không có code bị comment-out. Header `// id:` không có, nhưng toàn repo cũng chưa áp quy ước này, nên không tính là finding.

---

## 4c45595 — feat(cskh): fill code assignment status and campaign stock

**Phạm vi:** Thêm bảng `campaign_stock` (entity/repo/store/lookup REST pp-coupon `GET /api/v1/vouchers` + `configs/campaign/{id}`), service backfill + refresh live trên coupon-config event, endpoint `/campaign-stock`. Live path ghi `assignment_status`/`assigned_at` khi publish/revoke voucher. Sửa 2 lỗi map entity (`campaign_code` và `applies_to_all` int). 25 file, +508/−9.

**Đánh giá:** ⚠️ Cần sửa

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| 🟠 Major | `B/adapter/in/messaging/CouponDisplayProjectionConsumer.java:53` (chỗ gọi refresh duy nhất); `B/application/service/CampaignStockBackfillService.java:57-80` | `campaign_stock` **chỉ được refresh khi có coupon-config event**. Số `unassigned_code_count` giảm mỗi lần phát mã (voucher published) và đổi khi revoke, nhưng không có event nào trên luồng đó kích refresh. Ngoài ra pp-coupon sinh mã bất đồng bộ sau khi tạo config, nên lúc nhận `CouponConfigCreated` số đếm thường vẫn là 0. Hậu quả: nhãn "Còn mã" / "Chưa sinh mã" và `assignment_status` của bản ghi giả lập (`CustomerOfferQueryAdapter` UNASSIGNED_BRANCH, ở 5685a0c dòng ~104-111) bị sai kéo dài. Kiểm tra đến 5685a0c: `campaignStock.refresh` vẫn chỉ được gọi ở đúng một chỗ, **chưa sửa**. | Refresh thêm khi có voucher published/revoked/generated event (có debounce theo campaign), hoặc dùng job định kỳ có distributed lock (ShedLock) để làm mới các campaign RUNNING. Ít nhất phải ghi rõ độ trễ chấp nhận được trong DD. |
| 🟠 Major | `B/adapter/out/persistence/VoucherOwnershipStoreAdapter.java:32,110` | Live path ghi `assignment_status = 'revoked'`, nhưng từ vựng của cột (Javadoc entity `VoucherOwnershipEntity.java:112`: "published / unpublished / not_generated") và API spec (`CustomerOfferItem.assignmentStatus` allowableValues = `published, unpublished, not_generated`; `CustomerCareApi` filter cũng vậy) **không có `revoked`**. View `customer_voucher_view` không lọc dòng thu hồi, và `CustomerOfferQueryAdapter` (5685a0c:51) trả thẳng `v.assignment_status`, nên API trả ra giá trị nằm ngoài contract. FE map enum sẽ vỡ, và filter theo trạng thái không bao giờ chọn được những dòng này. | Thống nhất một trong hai cách: (a) bổ sung `revoked` vào spec/OpenAPI và DD, hoặc (b) ẩn dòng revoked ở query / map về giá trị hợp lệ. Đồng thời cập nhật Javadoc của entity. |
| 🟡 Minor | `B/adapter/in/messaging/CouponDisplayProjectionConsumer.java:50-57` | `ensureCouponTypeForCampaign` và `campaignStock.refresh` nằm chung một `try`. Lookup coupon_type ném lỗi thì bỏ luôn refresh tồn kho, dù hai việc độc lập nhau. | Tách hai `try` riêng, giống cách 0a20898 đã tách projection khỏi bước lấp. |
| 🟡 Minor | `B/application/service/CampaignStockBackfillService.java:57-80` + consumer | Mỗi coupon-config event giờ tạo tới khoảng 4 REST call đồng bộ (count total, count unassigned, config-by-campaign, shared code), cộng N call coupon_type, trong khi read-timeout Feign mặc định là 60s và các client `pp-coupon-config`/`pp-campaign-detail` không có circuit breaker. Khi pp-coupon chậm, một batch có thể vượt `max.poll.interval.ms`, dẫn tới rebalance và nhận lại batch liên tục. | Đặt read-timeout ngắn riêng cho 2 client này (vd 3-5s) + circuit breaker, hoặc đẩy việc refresh sang async/outbox. |
| 🟡 Minor | `B/application/service/CampaignStockBackfillService.java:38,45,62` | Ở chế độ chạy thật, `countUnassignedCodes` bị gọi 2 lần cho mỗi campaign (một lần ở vòng lặp, một lần trong `refresh`), tốn thêm một REST call trên mỗi campaign. | Cho `refresh` trả về kết quả (đã ghi hay chưa) và đếm `updated`/`skipped` dựa vào đó. Chỉ gọi `countUnassignedCodes` riêng khi dryRun. |
| 🟡 Minor | `B/adapter/out/persistence/CampaignStockStoreAdapter.java:24-41` | Upsert kiểu `findById` → `save`, không lock. Consumer và endpoint backfill chạy song song trên cùng campaign có thể cùng INSERT, gây lỗi trùng PK (`DataIntegrityViolationException`). Ở consumer lỗi này bị nuốt thành log.warn, ở backfill thì làm hỏng cả trang. Giá trị là snapshot tuyệt đối chứ không phải bộ đếm cộng dồn, nên không sai số, chỉ lỗi hoặc last-writer-wins. | Dùng native `INSERT ... ON DUPLICATE KEY UPDATE` (MariaDB), hoặc bắt lỗi trùng khoá rồi thử lại một lần. |
| 🟡 Minor | `B/adapter/out/external/CouponConfigFeignClient.java:46-57` | Method mới được chèn **giữa** Javadoc của `getConfigByCampaign` và chính method đó. Kết quả là hai khối `/** */` liền nhau: Javadoc cũ trôi sang `listVouchers`, còn `getConfigByCampaign` mất tài liệu. Import mới (dòng 5-6) cũng chen vào giữa nhóm import Spring. | Chuyển `listVouchers` xuống dưới `getConfigByCampaign` (hoặc lên trên khối Javadoc), sắp lại import. |
| 🟡 Minor | `B/application/port/out/CouponConfigLookupPort.java` (record `ConfigSummary`, `@param couponType {@code BULK} hoặc {@code SHARED}`); changeset 022 dòng 21 | Tài liệu ghi giá trị `BULK`/`SHARED`, nhưng code so với `SINGLE_CODE` (`CampaignStockBackfillService.java:84`) và API spec dùng `BULK_CODE`/`SINGLE_CODE`. Từ vựng không nhất quán, dễ dẫn tới so sai chuỗi. | Sửa Javadoc/comment theo đúng `BULK_CODE`/`SINGLE_CODE`. Changeset đã merge thì không sửa, chỉ ghi chú ở DD. |
| 🟡 Minor | `CampaignStockBackfillService`, `CampaignStockLookupAdapter`, `VoucherOwnershipStoreAdapter` (nhánh assignment) | Không có test nào cho logic mới: tính `sharedCode` cho SINGLE_CODE, giữ nguyên giá trị cũ khi lookup lỗi, ghi `assignment_status` khi publish/revoke. Test chỉ được sửa cho compile lại. Kiểm tra đến 5685a0c vẫn chưa có. | Bổ sung unit test cho service (mock port) và test adapter (mock Feign 200/rỗng/lỗi). |
| 🔵 Info | `B/adapter/out/external/CampaignStockLookupAdapter.java:49` | `Math.toIntExact` ném `ArithmeticException`, không nằm trong `catch (FeignException)`. Thực tế gần như không xảy ra vì không ai có hơn 2^31 mã, nhưng nếu xảy ra thì đi ngược tinh thần "best-effort". | Có thể bỏ qua, hoặc clamp / bắt thêm lỗi này. |
| 🔵 Info | `B/application/service/CampaignStockBackfillService.java:83-84` | Hằng `private static final` đặt cuối class, trái thứ tự khai báo thông thường. | Chuyển lên đầu class. |

**Ghi chú:**
- Hai fix mapping entity có giá trị thật: `campaignCode` trước đó map nhầm vào `@Column(name="campaign_type")`, gây trùng cột nên không khởi động được; `applies_to_all` qua view là `int`. Bảng `campaign_stock` và các cột mới của `voucher_ownership` đã có ở changeset 022/023 (41e1902), nên mapping khớp schema.
- Chọn hỏi thẳng pp-coupon lấy số tuyệt đối thay vì cộng trừ theo event là quyết định đúng: không trôi số, không cần atomic increment. Vấn đề nằm ở **tần suất refresh** (finding Major ở trên), không nằm ở cách đếm.
- `assigned_at` được tách khỏi `received_at`, không đổi ngữ nghĩa cột cũ đang phục vụ app khách hàng.
- Không có import thừa, không có code bị comment-out. S1192: `"campaign_stock"` chỉ xuất hiện trong log, chưa tới 3 lần mỗi file.

---

## b30d84d — feat(cskh): backfill code assignment status for existing rows

**Phạm vi:** Endpoint `/assignment-status` + service/store backfill `assignment_status`/`assigned_at` cho `voucher_ownership` cũ bằng 2 câu native `UPDATE ... WHERE assignment_status IS NULL LIMIT :limit` (revoked trước, published sau), gọi lặp đến khi `hasNext=false`. 6 file, +138 dòng.

**Đánh giá:** ⚠️ Cần sửa

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| 🟠 Major | `B/adapter/out/persistence/repository/VoucherOwnershipRepository.java:197-202` (`backfillAssignmentPublished`: `AND status <> 'REVOKED'`) | Cột `voucher_ownership.status` **cho phép NULL** (changeset 005 dòng 18: `VARCHAR(32)` không có `nullable=false`). Theo logic ba trị của SQL, `NULL <> 'REVOKED'` cho ra UNKNOWN, nên dòng có `status IS NULL` không khớp nhánh published lẫn nhánh revoked và **không bao giờ được lấp**. Hệ quả: (1) `countMissingAssignmentStatus` không bao giờ về 0; (2) `hasNext = missing > published + revoked` (`VoucherAssignmentBackfillService.java:25`) luôn true, người vận hành gọi lặp "tới khi hasNext=false" theo Javadoc controller sẽ **lặp vô hạn**; (3) API trả `assignment_status = null` cho các dòng này. | Đổi điều kiện thành `AND (status IS NULL OR status <> 'REVOKED')`. Tính `hasNext` dựa trên số dòng đã ghi trong lượt (`published + revoked == size`) hoặc đếm lại sau khi update, thay vì so với số đếm trước đó. |
| 🟡 Minor | `B/adapter/out/persistence/repository/VoucherOwnershipRepository.java:198,208` (`assigned_at = received_at`) | `received_at` nullable (changeset 005 dòng 22). Live path có fallback (`VoucherOwnershipStoreAdapter.java:76` ở 4c45595: `receivedAt != null ? receivedAt : now`), backfill thì không, nên dòng cũ thiếu `received_at` sẽ có `assigned_at = NULL`. Hai đường ghi xử lý không nhất quán. | `assigned_at = COALESCE(received_at, created_at)`. |
| 🟡 Minor | `B/adapter/out/persistence/repository/VoucherOwnershipRepository.java:187-213` | Cột `assignment_status` không có index. Mỗi lượt gọi chạy `COUNT(*) ... IS NULL` (full scan) cộng 2 câu `UPDATE ... LIMIT` quét từ đầu bảng, mà các dòng đã lấp ở lượt trước vẫn phải đọc qua, nên tổng chi phí xấp xỉ bậc hai theo kích thước bảng. Thêm nữa, `UPDATE ... LIMIT` không có `ORDER BY` bị MariaDB đánh dấu *unsafe* cho binlog dạng STATEMENT. | Lấp theo khoảng khoá chính (keyset: `WHERE id > :lastId ORDER BY id LIMIT :n`) để resumable mà không cần index. Nếu giữ cách hiện tại thì thêm `ORDER BY` theo PK. |
| 🟡 Minor | `B/application/service/VoucherAssignmentBackfillService.java:16-29` | Không có test cho service: nhánh dryRun, chia limit giữa revoked/published, công thức `hasNext`. Kiểm tra đến 5685a0c vẫn chưa có. | Bổ sung unit test (mock store), có case dòng `status NULL`. |
| 🔵 Info | `B/application/service/VoucherAssignmentBackfillService.java:25` | Ở chế độ dryRun, `hasNext = missing > size` chỉ là ước lượng, không phản ánh việc trang có ghi được hay không. Chấp nhận được với dry-run, nên ghi rõ trong Javadoc. | — |

**Ghi chú:**
- Thiết kế tốt: update có điều kiện `assignment_status IS NULL` nên **idempotent** và **không đè** giá trị live path đã ghi. Không phụ thuộc phân trang offset (tránh lỗi bỏ sót khi tập kết quả co lại như các backfill khác). Mỗi câu UPDATE là một transaction ngắn, không gọi REST bên trong.
- `@Modifying(clearAutomatically, flushAutomatically)` + `@Transactional` ở adapter là đúng chỗ.
- Endpoint mặc định `dryRun=true`, `size` được giới hạn `@Max(5000)`.
- Không có import thừa, không có code bị comment-out, không có literal lặp ≥3 lần.

---

## 1ff368a — feat(cskh): project offer history and usage status

**Phạm vi:** Thêm 2 consumer Kafka (`RedemptionAuditConsumer`, `CashbackAuditConsumer`) chiếu kết quả đổi ưu đãi và hoàn tiền vào bảng chỉ-thêm `offer_audit`, đồng thời cập nhật `voucher_ownership.usage_status` (có chặn event tới muộn bằng `last_event_at`) và bảng mới `cashback_participation` (changeset 024). Ghi thêm dòng lịch sử ASSIGN khi gán mã. 26 file, +1170/-4 (main 20 file +861, test 6 file).

**Đánh giá:** ⚠️ Cần sửa

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| 🟠 Major | `adapter/out/persistence/VoucherOwnershipStoreAdapter.java:347-353` + `application/service/OfferAuditProjectionService.java:64-80` | Gán nhầm khách: khi lọc theo `customerId` không ra dòng nào, `findOwners` thử lại **không lọc** và trả về MỌI chủ sở hữu của mã. Với mã chung (SINGLE_CODE), một phiên đổi của 1 khách sẽ ghi dòng REDEEM và set `usage_status` success/failed cho **tất cả** khách đang giữ mã đó. Màn CSKH sẽ hiện "đã sử dụng" cho những khách chưa từng dùng. | Chỉ fallback khi kết quả không lọc có **đúng 1** dòng (mã riêng). Nếu nhiều dòng thì ghi log (đã mask) rồi bỏ qua, hoặc đẩy vào hàng đợi đối soát. Không fallback cho `applyUsageStatus`. |
| 🟠 Major | `adapter/out/persistence/OfferAuditStoreAdapter.java:28-29` | Khoá dedupe `(eventId, voucherCode, action)` **không có `customerSourceId`**. Với cùng 1 event mà mã chung thuộc nhiều khách (đúng trường hợp comment ở `OfferAuditProjectionService.java:62-63` nói muốn "mỗi khách một dòng"), chỉ khách đầu tiên được ghi lịch sử, các khách sau bị bỏ qua vì "trùng". Test `shared_code_writes_one_row_per_owner` qua được chỉ vì store bị mock. | Đưa `customerSourceId` vào khoá dedupe (cả method repository và index), thêm test adapter hoặc test tích hợp cho trường hợp mã chung. |
| 🟠 Major | `adapter/out/persistence/OfferAuditStoreAdapter.java:28-48`, `adapter/out/persistence/CashbackParticipationStoreAdapter.java:24-46` | Idempotency dựa vào check-then-act (`existsBy...` rồi `save`, `findBy...` rồi `save`) mà **không có unique constraint** ở `offer_audit` (023 cố ý chỉ đặt index thường trên `event_id`). Event giao lại hoặc chạy song song (rebalance, nhiều instance) vẫn có thể ghi trùng lịch sử. Ở `cashback_participation` có `uk_cashback_customer_campaign` nhưng insert đồng thời sẽ ném `DataIntegrityViolationException`, rollback luôn dòng audit cùng transaction và bị consumer nuốt mất (xem dòng dưới). | Thêm unique index `(event_id, customer_source_id, voucher_code, action)` cho `offer_audit` (cần xử lý `voucher_code` null, ví dụ dùng cột sinh `COALESCE`) và dùng insert-ignore / bắt duplicate-key. Với participation dùng upsert nguyên tử (`INSERT ... ON DUPLICATE KEY UPDATE ... WHERE last_event_at < ?`). |
| 🟠 Major | `adapter/in/messaging/RedemptionAuditConsumer.java:41-46`, `adapter/in/messaging/CashbackAuditConsumer.java:43-48` | `catch (RuntimeException)` chỉ log rồi vẫn `ack.acknowledge()`: lỗi tạm thời (DB mất kết nối, deadlock, duplicate key do race) làm **mất vĩnh viễn** event lịch sử. Không retry, không DLT. Vi phạm rule "không catch nuốt exception". Pattern này chép từ các consumer chiếu cũ, nhưng với audit trail CSKH (append-only, không có job đối soát) thì mất dữ liệu là không phục hồi được. | Chỉ bắt lỗi dữ liệu không xử lý được (poison). Lỗi hạ tầng thì để container retry (`DefaultErrorHandler` + backoff + `DeadLetterPublishingRecoverer`), hoặc ít nhất đẩy vào DLT trước khi ack. **Chưa sửa** ở các commit sau. |
| 🟠 Major | `adapter/out/persistence/repository/VoucherOwnershipRepository.java:217-222` (gọi từ `VoucherOwnershipStoreAdapter.java:343,350`) | `findOwners` lọc theo `voucher_code`, nhưng `voucher_ownership` **không có index trên `voucher_code`** (005 chỉ có `customer_source_id`, `campaign_id`, unique `(customer_source_id, voucher_id)`; 014 chỉ có index `voucher_id`; tới 5685a0c vẫn chưa thêm). Mỗi mã trong mỗi event redemption gây 1 đến 2 lần full scan, và `applyUsageStatus` (line 240) cũng phải scan theo `voucher_code`. | Thêm changeset mới tạo index `(voucher_code, customer_id)` hoặc `(voucher_code)` trên `voucher_ownership`. |
| 🟠 Major | `application/service/CashbackAuditProjectionService.java:38`, `:60` | `log.warn("... {}", event)` dump nguyên object event. `CashbackSuccessEventPayload` mang `customerPhone` (msisdn) và `customerId`, nên nếu payload có `toString` sinh tự động (Lombok, vì có builder và setter) thì msisdn bị log thô ở mức WARN. *(Chưa đọc được class trong jar để xác nhận `toString`.)* | Chỉ log `event.getId()`, `getType()` và `transactionId`. Không dump object. |
| 🟡 Minor | `application/service/OfferAuditProjectionService.java:81-82`, `adapter/out/persistence/CashbackParticipationStoreAdapter.java:35-36` | Log debug in `customerSourceId` (msisdn) chưa mask. | Mask (vd `849****525`) hoặc bỏ. |
| 🟡 Minor | `application/service/CashbackAuditProjectionService.java:64-68` + `:86-91`, `adapter/out/persistence/CashbackParticipationStoreAdapter.java:53-57` | `CashbackFailedEvent` không có msisdn nên phải tra từ `cashback_participation` theo `customerId`. (1) Khách thất bại ngay lần đầu (chưa có dòng participation) thì event bị **bỏ hẳn**, CSKH không thấy giao dịch thất bại. (2) `findByCustomerId(...).findFirst()` không có ORDER BY, nên khách có nhiều thuê bao sẽ nhận msisdn tuỳ ý. | Fallback tra `voucher_ownership.customer_id` (hoặc nguồn định danh khách khác). Ưu tiên dòng cùng `campaignId`, sắp theo `updated_at DESC`. Khi vẫn không ra msisdn thì ghi vào hàng đợi chờ thay vì drop. |
| 🟡 Minor | `application/service/OfferAuditProjectionService.java:53`, `:96`, `application/service/CashbackAuditProjectionService.java:101` | Event thiếu mốc thời gian thì lấy `Instant.now()` làm mốc nghiệp vụ. Event cũ phát lại sẽ mang mốc "mới nhất", vượt qua chốt `last_event_at` và kéo lùi trạng thái, đúng điều mà thiết kế muốn chặn. | Thiếu mốc thì vẫn ghi audit nhưng KHÔNG cập nhật `usage_status`/participation, hoặc gắn cờ cần đối soát. |
| 🟡 Minor | `adapter/out/persistence/repository/VoucherOwnershipRepository.java:240`, `adapter/out/persistence/CashbackParticipationStoreAdapter.java:33-34` | Chốt so sánh nghiêm ngặt (`<` / `!isAfter`): hai event khác nhau cùng mili-giây (vd 2 giao dịch hoàn tiền cùng lúc) thì event sau bị xem là "tới muộn". Ngoài ra lần thử thất bại sau một lần dùng thành công (vd "Mã đã hết lượt") sẽ ghi đè `usage_status` từ `success` sang `failed`. Cần xác nhận với SRS 4.4 xem đó có phải ý đồ nghiệp vụ không. | Tie-break bằng event id/transaction id. Làm rõ quy tắc chuyển trạng thái sử dụng (state machine tường minh: success có bị failed ghi đè không). |
| 🟡 Minor | `src/test/.../OfferAuditProjectionServiceTest.java`, `CashbackAuditProjectionServiceTest.java` | Test chỉ là unit test service với store mock. Chưa test: SQL `applyUsageStatus` (chốt late-event), `findOwners` fallback, dedupe `existsBy...` với `voucher_code` null, `CashbackParticipationStoreAdapter.upsert` (nhánh late-event), 2 consumer (cô lập lỗi, nhánh `default`), và việc `CustomerVoucherProjectionService` gọi `recordAssignment` (5 test chỉ inject mock, không verify). | Bổ sung test adapter (`@DataJpaTest`/Testcontainers MySQL) và verify `recordAssignment` trong `CustomerVoucherProjectionServiceTest`. |
| 🔵 Info | `src/main/resources/db/changelog/changes/024-cashback-participation.xml:18`, `:47` | `author="claude"` trong changeset (giống 022/023). | Đổi sang tài khoản dev thật để audit trail migration rõ người chịu trách nhiệm. |
| 🔵 Info | Mọi file `.java` mới | Thiếu header `// id: ... \| dd_version: vN`. Toàn repo hiện cũng chưa file nào có header, nên đây là lệch quy ước dự án chứ không phải riêng commit này. | Bổ sung theo Rule 1 nếu repo áp pre-commit hook NTAPP. |

**Ghi chú:**
- Điểm tốt: tách rõ "audit luôn ghi" và "trạng thái chỉ ghi khi mới hơn". Chốt late-event làm bằng UPDATE có điều kiện nguyên tử (`last_event_at IS NULL OR last_event_at < :occurredAt`), không read-modify-write. Tiền dùng `BigDecimal` + `DECIMAL(18,4)`, thời gian dùng `Instant`. Cắt `note` để insert không lỗi. Lỗi được cô lập từng event trong lô. Migration 024 là file mới, không sửa changeset cũ, có rollback, và có unique `(customer_source_id, campaign_id)`.
- Sonar: S125 (grep `//...;$` không ra dòng nào), S1128 (không có import thừa), S1192 (literal đã tách constant), S3776 (method lớn nhất `applyRedemptionResult` khoảng 12 điểm, dưới ngưỡng): đều không vi phạm.
- Không có gọi API ngoài hay gửi message trong `@Transactional`. `recordAssignment` chạy chung transaction với upsert ownership: audit lỗi thì upsert cũng rollback, chấp nhận được.
- Commit sau 90cc652 chỉ đổi từ vựng action/result (`SESSION_CONFIRMED`, `VOUCHER_ASSIGNED`, `SUCCESS`/`FAILED`). Các finding Major ở trên (fallback gán nhầm khách, khoá dedupe, nuốt exception, thiếu index `voucher_code`, log dump event) **chưa được sửa** tới đầu nhánh 5685a0c.

---

## 489122a — feat(cskh): implement customer offer search

**Phạm vi:** Triển khai `GET /api/cskh/v1/customer-offers`: controller uỷ quyền cho `CustomerCareSearchService`, adapter native SQL `UNION ALL` hai nhánh (voucher view + cashback_participation) rồi lọc, đếm, phân trang, đánh dấu `hasHistory` bằng một câu IN. 7 file, +418/-3.

**Đánh giá:** ⚠️ Cần sửa (blocker xác thực đã loại khỏi phạm vi)

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| ⚪ Dismissed (ngoài phạm vi — auth CSKH triển khai theo hướng khác) | `…/web/CustomerCareController.java:47-50, 95-104` | Commit này nối endpoint vào dữ liệu thật trong khi Javadoc ngay trên class ghi rõ vùng `/api/cskh/v1` **chưa có filter xác thực nào** (`VtmAuthFilter` chỉ lọc `/api/v1/`). Ai gọi tới được BFF cũng tra được ưu đãi, mã voucher, lịch sử của **bất kỳ số thuê bao nào** mà không cần token (IDOR/lộ PII). Không có `@PreAuthorize` hay check role CSKH. Tới 5685a0c vẫn chưa sửa, cảnh báo vẫn còn ở dòng 52. | Thêm filter xác thực token nhân viên (Keycloak) cho `/api/cskh/v1/**` và `@PreAuthorize` với role CSKH khai báo tập trung. Cần có test 401/403 trước khi merge vào staging. |
| 🟠 Major | `…/persistence/CustomerOfferQueryAdapter.java:104-105` | Thứ tự sắp xếp lệch spec. Spec (`CustomerCareApi` Javadoc, SRS 2.3): ngày bắt đầu chiến dịch DESC → ngày gán DESC → ngày sử dụng DESC → mã chiến dịch ASC. Code đang sort `campaign_start_date DESC, campaign_id, voucher_code`. Chưa sửa ở commit sau. | `ORDER BY t.campaign_start_date DESC, t.assigned_at DESC, t.used_at DESC, t.campaign_code ASC`, rồi thêm `t.campaign_id, t.voucher_code` làm tie-breaker để phân trang ổn định. |
| 🟠 Major | `…/persistence/CustomerOfferQueryAdapter.java:149-152` | Lọc `productCode` bằng `applicable_products LIKE '%X%'`. Theo cc01d14, cột này là **mảng JSON** (có id nội bộ, mã nguồn, ảnh, và cả item `EXCLUDED`). Hậu quả: (1) chiến dịch **loại trừ** sản phẩm X vẫn khớp khi lọc theo X, (2) khớp chuỗi con (`SKU1` khớp `SKU10`, khớp cả tên/ảnh). Chưa sửa ở commit sau, cc01d14 chỉ sửa phần đọc. | Dùng `JSON_CONTAINS`/`JSON_SEARCH` (MariaDB) trên đúng field mã sản phẩm và loại item EXCLUDED, hoặc chuẩn hoá ra bảng con `campaign_product(campaign_id, product_code, type)`. |
| 🟠 Major | `…/app/service/CustomerCareSearchService.java:42-49` | `splitProducts` cắt chuỗi theo dấu phẩy, trong khi read model lưu JSON, nên `applicableProducts` trả ra các mảnh JSON vỡ. | **Đã sửa ở cc01d14** (`ApplicableProductScope`). |
| 🟡 Minor | `…/web/api/CustomerCareApi.java:121,136,152,162` (contract) + adapter `:133-148` | Các tham số enum (`assignmentStatus`, `campaignStatus`, `usageStatus`, `offerType`) chỉ có `@Size(max=32)`, không whitelist. Giá trị lạ không trả 400 mà thành điều kiện `=` luôn rỗng. Rule BE §4 yêu cầu parse về enum. | Khai enum Java (hoặc `@Pattern` với tập giá trị) để trả 400 đúng spec. Map enum sang giá trị cột một cách tường minh. |
| 🟡 Minor | `…/persistence/CustomerOfferQueryAdapter.java:107` (+ `OfferAuditQueryAdapter:104` ở 90cc652) | `q.page() * q.size()` tính bằng `int`. `page` chỉ có `@Min(0)`, không có max, nên với `page=21474837&size=100` phép nhân tràn số âm, Hibernate ném `IllegalArgumentException` và trả 500. | Thêm `@Max` cho `page`, hoặc tính offset bằng `Math.multiplyExact`/`long` rồi chặn lại. |
| 🟡 Minor | `…/persistence/CustomerOfferQueryAdapter.java:126-131, 149-151` | `campaignName`/`productCode` không có `@Pattern`, ký tự `%` và `_` của người dùng đi thẳng vào LIKE thành wildcard (không phải SQL injection vì đã bind tham số). `voucherCode` thì đã an toàn nhờ `^[a-zA-Z0-9]*$`. | Escape `%`, `_`, `\` trước khi bọc `%…%` (thêm `ESCAPE '\\'`). |
| 🟡 Minor | `…/persistence/CustomerOfferQueryAdapter.java:126-131` | Spec ghi tên chiến dịch "bỏ dấu", code chưa bỏ dấu (comment thừa nhận là giới hạn). Nhánh cashback dùng `campaign_code` làm tên, nên tìm theo tên thật của chiến dịch hoàn tiền sẽ không ra. | Ghi nhận là nợ kỹ thuật/đổi mô tả spec cho khớp, hoặc chuẩn hoá cột normalized (bỏ dấu) ở phía ghi. |
| 🟡 Minor | `…/persistence/CustomerOfferQueryAdapter.java:37-56` | Nhánh voucher không loại các dòng `assignment_status='revoked'` (ghi bởi `VoucherOwnershipStoreAdapter.markRevoked`), nên API trả giá trị `revoked`, không nằm trong `allowableValues` của `CustomerOfferItem.assignmentStatus`. | Hoặc loại dòng revoked khỏi kết quả tra cứu, hoặc bổ sung `revoked` vào spec/enum. |
| 🟡 Minor | `src/test/.../CustomerCareSearchServiceTest.java` | Test chỉ mock port, nên toàn bộ SQL động (`buildWhere`, UNION, sort, phân trang, `markHistory`) không có test nào chạy trên DB thật. | Thêm integration test (Testcontainers MariaDB) cho adapter: từng filter, sort, trang cuối, `hasHistory`. |

**Ghi chú:** Bind tham số đầy đủ, không nối input vào SQL, ORDER BY cố định nên không có rủi ro injection. Lọc và phân trang làm trên tập đã UNION là đúng (Javadoc giải thích rõ). `markHistory` gom một câu IN cho cả trang, tránh N+1. Không log msisdn. Lọc theo msisdn có index (`idx_ownership_customer_campaign`), nên `LIKE '%…%'` chỉ quét trong phạm vi một khách và chấp nhận được. MariaDB collation ci nên so sánh `campaign_status='running'` với `RUNNING` vẫn khớp.

---

## d2df587 — feat(cskh): add offers the customer has no code for

**Phạm vi:** Thêm nhánh thứ 3 `UNASSIGNED_BRANCH` vào UNION: mỗi chiến dịch đang `RUNNING` mà khách chưa có mã/chưa tham gia sinh một bản ghi giả lập, với nhãn "Còn mã" / "-" / mã chung lấy từ `campaign_stock`. 1 file, +51/-3.

**Đánh giá:** ⚠️ Cần sửa

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| 🟠 Major | `…/persistence/CustomerOfferQueryAdapter.java:90-92, 122-129` | Không lọc theo segment/điều kiện tham gia (comment và commit message tự thừa nhận). Hậu quả: (1) **mọi** số thuê bao, kể cả số không tồn tại, đều thấy toàn bộ chiến dịch RUNNING kèm nhãn "Còn mã", nên CSKH có thể báo sai cho khách rằng khách được hưởng ưu đãi. (2) Vi phạm contract: spec ghi "Khách hàng chưa được phát ưu đãi nào trả về trang rỗng, không phải lỗi" (`CustomerCareApi` mô tả operation), nhưng giờ không bao giờ rỗng khi có chiến dịch đang chạy. Chưa sửa ở commit sau. | Cần PO/BA chốt. Hoặc chỉ hiện nhánh này khi có dữ liệu đủ điều kiện (segment/eligibility), hoặc thêm cờ/nhãn "chưa xác định điều kiện" và cập nhật spec. Ít nhất phải sửa mô tả spec cho khớp hành vi. |
| 🟠 Major | `…/persistence/CustomerOfferQueryAdapter.java:112, 116` | `JOIN campaign_stock` là **INNER JOIN**. `campaign_stock` chỉ được lấp từ coupon config (`CouponDisplayProjectionConsumer`, `CampaignStockBackfillService` lookup pp-coupon), nên chiến dịch **Hoàn tiền** không có dòng stock và bị loại hẳn. Nhánh `'not_participated'` (dòng 112) vì vậy là code chết, giá trị `not_participated` trong spec không bao giờ được trả. Chiến dịch voucher chưa kịp backfill stock cũng biến mất mà không báo gì. | Đổi sang `LEFT JOIN campaign_stock` và xử lý `s.*` NULL. Với `campaign_type='CASHBACK'` phải trả `voucher_code`/`assignment_status`/`coupon_type` = NULL theo spec (hiện CASE dòng 104-110 không xét CASHBACK, nếu có stock sẽ ra `not_generated`). |
| 🟡 Minor | `…/persistence/CustomerOfferQueryAdapter.java:117-120` | Derived table `GROUP BY campaign_id` trên toàn bộ `coupon_display` được tính lại ở **cả** câu COUNT lẫn câu lấy trang, ở mỗi request. Quét toàn bảng, tốn tài nguyên khi `coupon_display` lớn. `MIN(title)` cũng chọn tên tuỳ ý khi một chiến dịch có nhiều coupon config. | Đẩy điều kiện `campaign_id IN (chiến dịch RUNNING)` vào trong derived table, hoặc lưu sẵn title/coupon_type ở `campaign_offer` (đã có cột `campaign_code` theo cùng pattern). |
| 🟡 Minor | (không có file test) | Nhánh mới với logic CASE nhãn ("Còn mã", "-", mã chung, `not_generated`) không có test nào. | Thêm integration test cho 4 tình huống tồn kho + chiến dịch cashback. |

**Ghi chú:** `NOT EXISTS` trên `voucher_ownership(customer_source_id, campaign_id)` có index hỗ trợ. Gộp chuỗi union thành biến `union` giúp COUNT và SELECT dùng chung một định nghĩa (bớt rủi ro lệch). Sonar: không có vi phạm (literal SQL nằm trong text block).

---

## fc62ac2 — feat(cskh): match msisdn typed as 0xxx or 84xxx

**Phạm vi:** Thêm `MsisdnVariants.of()` sinh hai dạng 0xxx/84xxx. Mọi điều kiện `customer_source_id = :msisdn` trong adapter tra cứu (3 nhánh + `markHistory`) đổi sang `IN (:msisdns)`. Cập nhật mô tả param và thêm 7 unit test. 4 file, +128/-8.

**Đánh giá:** ✅ OK

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| 🟡 Minor | `…/web/api/CustomerCareApi.java:198` (tại 90cc652, vẫn còn ở 5685a0c:236) | API audit (90cc652) cũng dùng `MsisdnVariants`, nhưng mô tả param `msisdn` của operation audit vẫn ghi "khớp chính xác". Contract lệch hành vi. | Đổi mô tả giống operation tra cứu ("Nhận cả hai dạng 0xxx và 84xxx…"). |
| 🔵 Info | `…/app/support/MsisdnVariants.java:36-58` | Không kiểm độ dài số thuê bao VN (0 + 9 số / 84 + 9 số), nên chuỗi số bất kỳ có đầu 0/84 đều sinh biến thể (vd `012` → `8412`). Dạng 9 số không đầu (`328419491`) không được xử lý. Rủi ro thấp vì API đã chặn `^[0-9]{1,15}$`. | Cân nhắc chỉ sinh biến thể khi độ dài khớp số di động VN, hoặc chuẩn hoá msisdn một lần ở phía ghi read model để bỏ hẳn nhu cầu tra nhiều dạng. |
| 🔵 Info | `…/app/support/MsisdnVariants.java:48, 54` | Comment `// 84xxx -> 0xxx…` và `// 0xxx -> 84xxx` có token `->` (dạng lambda). Theo checklist S125 nội bộ đây là token rủi ro, grep `//.*;$` thì sạch. | Diễn đạt bằng lời ("đổi đầu 84 sang đầu 0") để chắc chắn không bị S125. |

**Ghi chú:** `IN (:msisdns)` vẫn dùng được index. Giữ nguyên định danh không phải số (UUID) và chặn sinh `00…`, test bao đủ biên (null/blank, chỉ có tiền tố, UUID, `840…`). `NOT EXISTS` của nhánh giả lập cũng đã đổi sang `IN`, nên không lọt chiến dịch khách đã có mã ở dạng số còn lại. Đây là điểm dễ sót nhưng đã làm đúng.

---

## 90cc652 — feat(cskh): implement offer audit trail lookup

**Phạm vi:** Triển khai `GET /api/cskh/v1/customer-offers/audit`: kiểm `to >= from` (400 qua `InvalidParamsExceptionHandler`), 404 khi không có chiến dịch, `OfferAuditQueryAdapter` native SQL đếm và phân trang `offer_audit`. Đổi từ vựng action/result đang ghi sang đúng spec (`VOUCHER_ASSIGNED`, `SESSION_CONFIRMED`, `CASHBACK_CREATED`, `SUCCESS`/`FAILED`). 14 file, +348/-26.

**Đánh giá:** ⚠️ Cần sửa (nhẹ; blocker xác thực đã loại khỏi phạm vi)

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| ⚪ Dismissed (ngoài phạm vi — auth CSKH triển khai theo hướng khác) | `…/web/CustomerCareController.java:112-120` | Endpoint thứ hai trả dữ liệu thật (lịch sử đổi thưởng, mã giao dịch hoàn tiền, ghi chú lỗi của khách) mà vùng `/api/cskh/v1` vẫn chưa có xác thực/phân quyền. Cùng gốc với blocker ở 489122a, chưa sửa tới 5685a0c. | Như 489122a: filter xác thực nhân viên + `@PreAuthorize` role CSKH, có test 401/403. |
| 🟡 Minor | `…/app/service/CashbackAuditProjectionService.java:25-28`, `…/app/service/OfferAuditProjectionService.java:30-34` | Đổi giá trị lưu (`CASHBACK`→`CASHBACK_CREATED`, `REDEEM`→`SESSION_CONFIRMED`, `success`→`SUCCESS`…) nhưng không có migration/UPDATE cho các dòng `offer_audit` đã ghi bởi 1ff368a. Môi trường nào đã chạy nhánh này (dev) sẽ có từ vựng lẫn lộn, API trả giá trị ngoài `allowableValues`. | Thêm changeset Liquibase UPDATE ánh xạ giá trị cũ sang mới, hoặc xác nhận chưa môi trường nào có dữ liệu cũ. |
| 🟡 Minor | `…/app/service/CashbackAuditProjectionService.java:73, 81` | Giao dịch hoàn tiền **thất bại** cũng ghi action `CASHBACK_CREATED` (kèm `FAILED`). Nghĩa sai: chưa có khoản hoàn tiền nào được tạo. Spec có sẵn `TRANSACTION_QUALIFIED`/`TRANSACTION_CONFIRMED`. | Chọn action theo loại sự kiện (vd thất bại thì ghi `TRANSACTION_CONFIRMED` + `FAILED`), chốt lại với BA. |
| 🟡 Minor | `…/app/port/in/CustomerCareServicePort.java:27-32` | Chèn `AuditQuery` vào giữa, làm Javadoc của `SearchQuery` (dòng 27-31) bị mồ côi: hai khối Javadoc liền nhau, `SearchQuery` mất tài liệu. Vẫn còn ở 5685a0c. | Chuyển `AuditQuery` (kèm Javadoc của nó) xuống dưới `SearchQuery`. |
| 🟡 Minor | `…/web/CustomerCareController.java:37-39, 62` | Sau commit này hằng `NOT_IMPLEMENTED` không còn được dùng (Sonar S1068 unused private field), Javadoc class vẫn ghi "hai thao tác tra cứu… trả 501". | **Đã sửa ở 5685a0c** (hằng bị xoá, Javadoc viết lại). |
| 🟡 Minor | `…/web/InvalidParamsExceptionHandler.java:19` | Import `vn.viettel…InvalidTimeRangeException` chen giữa khối `org.springframework.*`, sai thứ tự import (style). | Organize imports. |
| 🟡 Minor | `src/test/.../CustomerCareSearchServiceTest.java` (+3 test) | Test service đủ 3 nhánh (có dữ liệu / 404 / rỗng), nhưng adapter SQL (filter voucherCode, session, khoảng from/to, sort, phân trang) và luồng controller (`to < from` thành 400 đúng khuôn `INVALID_PARAMS`) không có test. | Thêm WebMvc test cho 400/404 và integration test cho `OfferAuditQueryAdapter`. |

**Ghi chú:** HTTP status đúng spec: 400 cho `to < from` đi qua handler chung (cùng khuôn lỗi validate), 404 qua `CampaignNotFoundException` → `ResourceNotFoundException` với mã `PP-BFF-VTM-1004`. Query có index phù hợp (`idx_audit_customer_campaign_time` khớp `customer_source_id IN … AND campaign_id = … ORDER BY occurred_at DESC`). Bind tham số đầy đủ, `ORDER BY` cố định, dùng `Instant` xuyên suốt, không log PII trong code mới. Dùng chung `MsisdnVariants` với API tra cứu nên hai màn hình nhất quán. Gỡ lớp dịch từ vựng ở phía đọc bằng cách ghi đúng spec ngay từ projection là hướng gọn. Sonar S3776/S1192/S125/S1128: không vi phạm trong code mới.

---

## cc01d14 — fix: applicableProducts CSKH trả object thay vì chuỗi cắt sai

**Phạm vi:** Thay `splitProducts` (cắt chuỗi theo dấu phẩy, vốn sai vì cột `applicable_products` lưu mảng JSON) bằng component mới `ApplicableProductScope`. Component này parse JSON thành `ApplicableProduct` (dùng mã nguồn làm id, đổi ảnh sang URL preview, bỏ item EXCLUDED). Đổi kiểu `applicableProducts` ở 3 DTO CSKH từ `List<String>` sang `List<ApplicableProduct>`. Tổng 7 file, +197/-28.

**Đánh giá:** ⚠️ Cần sửa

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| 🟠 Major | `application/support/ApplicableProductScope.java:42-61` và `dto/cskh/CustomerOfferItem.java:45`, `AuditCampaignSummary.java:38` | Spec công bố "applicableProducts rỗng nghĩa là không giới hạn sản phẩm", nhưng `from()` cũng trả `[]` trong 2 trường hợp khác: (a) JSON hỏng, bị nuốt ở dòng 58-61; (b) chiến dịch chỉ khai sản phẩm loại trừ, vì toàn bộ item EXCLUDED bị lọc ở dòng 47. Với (b), CSKH sẽ trả lời khách là "áp dụng mọi sản phẩm" trong khi thực tế có sản phẩm bị loại. Đây là sai nghiệp vụ, không chỉ sai hiển thị. | Trả thêm `excludedProducts` (hoặc cờ `appliesToAll`/`hasExclusions`) để client không phải đoán. Với JSON hỏng, cân nhắc trả null hoặc cờ lỗi thay vì `[]`. Nếu giữ `[]` thì sửa mô tả @Schema cho đúng. |
| 🟡 Minor | `ApplicableProductScope.java:16-21` và `adapter/out/persistence/CustomerVoucherViewMapper.java:243-258` | Javadoc nói "đúng khuôn mà ví VTM đang trả", "giống hệt cách luồng ví làm". Thực tế luồng ví còn bung cả danh mục dịch vụ khi `applies_to_all` + `APPLY_TO_ORDER` (PROM-1401), CSKH thì không. Cùng một chiến dịch, app khách thấy danh sách đầy đủ còn CSKH thấy `[]`. Ngoài ra lambda map tại dòng 47-51 chép nguyên từ `CustomerVoucherViewMapper:253-257`, nên có 2 bản parse `StoredProduct` cần giữ đồng bộ bằng tay. | Cho `CustomerVoucherViewMapper` dùng lại `ApplicableProductScope` cho phần lọc + map. Hoặc ghi rõ trong javadoc rằng CSKH cố ý không bung danh mục. |
| 🟡 Minor | `ApplicableProductScope.java:58-59` | `catch (Exception ex)` quá rộng: nuốt cả lỗi lập trình (NPE trong resolver...), trong khi mapper cũ chỉ bắt `JsonProcessingException`. Log cũng không kèm campaignId nên không trace được bản ghi hỏng. | Chỉ bắt `JsonProcessingException`. Truyền thêm ngữ cảnh (campaignId) vào log, hoặc để nơi gọi tự log. |
| 🔵 Info | `service/OfferAuditProjectionService.java:78-80,102-104`, `CashbackAuditProjectionService.java:80-81`, `persistence/OfferAuditStoreAdapter.java:41` | `AuditEntry.applicableProducts` luôn rỗng, vì mọi writer audit đều truyền `productScope = null` (test mới cũng assert `isEmpty()`). Nếu sau này ghi JSON vào thì cột `product_scope` (1000 ký tự) bị `truncate` cắt cụt, JSON hỏng, rồi lại thành `[]` mà không báo gì. | Nếu trường này chưa có nguồn dữ liệu thì ghi rõ trong spec. Khi bắt đầu ghi JSON thì nâng cột lên TEXT, không truncate JSON. |
| 🔵 Info | 3 DTO CSKH | Contract đổi kiểu `List<String>` sang `List<object>`. Chưa release nên chấp nhận được, nhưng spec đã giao Bankplus tích hợp trước từ 71f689e (stub 501). | Báo phía Bankplus về thay đổi schema. |
| 🔵 Info | `persistence/CustomerOfferQueryAdapter.java:198-200` (có từ trước, 489122a) | Lọc `productCode` bằng `LIKE '%code%'` trên chuỗi JSON thô, nên khớp cả item EXCLUDED. Kết quả là offer trả về có thể không chứa sản phẩm đang lọc (sau khi commit này bỏ EXCLUDED). Ngoài ra `%`/`_` trong input không được escape. | Theo dõi ở commit 489122a. |
| 🔵 Info | Mọi file trong commit | Không có header `// id: ... \| dd_version: ...`. Cả repo đều chưa có (0/377 file), nên lệch quy ước NTAPP chứ không riêng commit này. | Thống nhất với team nếu pre-commit hook được bật cho repo này. |

**Ghi chú:**
- Sửa đúng bug gốc: dữ liệu là JSON nhưng code cũ cắt theo dấu phẩy, nên trả ra các mảnh JSON vụn.
- Test tốt: `ApplicableProductScopeTest` phủ đủ các nhánh: khuôn đầy đủ, id lấy theo source, URL ảnh, bỏ EXCLUDED, null/blank/JSON hỏng/chuỗi CSV cũ. `CustomerCareSearchServiceTest` cập nhật fixture JSON thật.
- Đổi `static` thành method instance là hợp lý. Sonar: S1128/S125/S1192/S3776 không vi phạm.

---

## 5685a0c — feat(cskh): API gộp chi tiết chiến dịch mã giảm giá

**Phạm vi:** Thêm `GET /api/cskh/v1/campaign-details/{campaignId}`. Endpoint gọi pp-campaign trước (nguồn chính), sau đó fan-out song song 4 nguồn phụ (pp-coupon, pp-pricing-engine, pp-validation bindings, app-config), rồi chạy đợt 3 song song (pp-product phân giải tên + cây điều kiện). Các kết quả được gộp qua 6 mapper thành `CampaignDetailResponse`, kèm khối `SourceAvailability`. Thêm executor riêng `campaignDetailExecutor`, 4 port + 4 adapter, DTO mới và 2 file test. Tổng 45 file, +2780/-38.

**Đánh giá:** ⚠️ Cần sửa

**Findings:**

| Mức | File:line | Vấn đề | Đề xuất |
|---|---|---|---|
| 🟠 Major | `adapter/config/BffAsyncConfig.java:55-65` + `service/CampaignDetailAggregationService.java:142-143` | Executor 4-8 luồng, queue 50, dùng rejection policy mặc định (`AbortPolicy`). Mỗi request nộp 6 tác vụ, nên khoảng 9-10 request đồng thời với downstream chậm là đầy 58 slot. Khi đó `CompletableFuture.supplyAsync` ném `RejectedExecutionException` **đồng bộ** ở luồng request. Lỗi này không đi qua `.exceptionally()`, nên cả lời gọi thành HTTP 500 generic, trái với lời hứa "nguồn phụ best-effort" và không phải 502/503 như spec. | Bọc `supplyAsync` trong try/catch `RejectedExecutionException` để trả fallback (đánh dấu nguồn không sẵn sàng), hoặc set `CallerRunsPolicy` / bulkhead. Viết test cho tình huống pool đầy. |
| 🟠 Major | `CampaignDetailAggregationService.java:93-94,127-128`; 4 adapter mới (`CampaignDetailViewAdapter`, `CouponConfigViewAdapter`, `DiscountFormulaViewAdapter`, `ValidationRuleViewAdapter`); `application.properties:395` | `join()` không có timeout. Các adapter mới không có `@CircuitBreaker`/`@Retry` (khác `RedemptionApiAdapter`), trong khi Feign `read-timeout=60000`. Trường hợp xấu nhất, một request treo khoảng 3 x 60s (nguồn chính, rồi đợt 2, rồi đợt 3) và giữ luồng pool suốt thời gian đó, làm nặng thêm finding trên. Downstream lỗi liên tục cũng không bị ngắt mạch. | Dùng `orTimeout`/`completeOnTimeout(fallback)` cho từng tác vụ phụ, với ngân sách tổng khoảng 3-5s. Cấu hình read-timeout riêng cho các client `pp-*-detail`. Thêm circuit breaker cho 4 adapter (nguồn chính map `CallNotPermittedException` về 502, handler đã có sẵn). |
| 🟠 Major | `CampaignDetailAggregationService.java:104-108` vs `dto/cskh/campaign/SourceAvailability.java:8-12`; `DiscountFormulaViewAdapter.java:42-45`; `ValidationRuleViewAdapter.java:40-47`; `CouponConfigViewAdapter.java:31-35` | Hợp đồng `SourceAvailability` ghi rõ "true nghĩa là nguồn đã trả lời được, **kể cả khi** chiến dịch chưa khai phần đó". Code thì set cờ bằng `config.isPresent()`, `formula.isPresent()`, `!bindings.isEmpty()`, mà adapter lại gộp "không có dữ liệu" (danh sách rỗng / 404) và "dịch vụ lỗi" về cùng một giá trị `Optional.empty()`/`List.of()`. Hệ quả: chiến dịch không gắn rule hoặc không có công thức sẽ báo `validationAvailable=false`/`discountAvailable=false`. Đúng thứ khối này sinh ra để chống lại là CSKH đọc dữ liệu nghiệp vụ thành sự cố hạ tầng. Javadoc `DiscountFormulaViewAdapter:19-22` còn nói chọn API lô **để** phân biệt được 2 nghĩa, nhưng code lại xoá mất sự phân biệt đó. Ngoài ra lỗi của cây điều kiện và app-config không phản ánh vào khối này. Test `secondaryFailuresDegradeGracefully` (dòng 121-134) đang khoá cứng hành vi sai: mock trả `List.of()`/`Optional.empty()` (tức là "không có dữ liệu") rồi assert `false`. | Cho adapter trả kiểu 3 trạng thái, ví dụ `SourceResult<T>{available, value}`, hoặc ném lỗi riêng để `secondary()` hạ cờ. Thêm cờ cho conditions. Viết test tách riêng "nguồn trả rỗng" (cờ true) và "nguồn lỗi" (cờ false). |
| ⚪ Dismissed (ngoài phạm vi — auth CSKH triển khai theo hướng khác) | `adapter/in/web/CustomerCareController.java:52-55` | Javadoc tự cảnh báo `/api/cskh/v1` CHƯA có filter xác thực (`VtmAuthFilter` chỉ lọc `/api/v1/`). Commit thêm một endpoint không xác thực nữa, trả cấu hình chiến dịch nội bộ (ngân sách, rule, tham số sinh mã, mã chung `sharedCode`/`customCode`, createdBy/updatedBy). Header Authorization của request vẫn được forward sang 4 downstream. | Bổ sung filter xác thực nhân viên + `@PreAuthorize` theo role CSKH trước khi merge lên môi trường có dữ liệu thật. Cân nhắc che `sharedCode`. |
| 🟡 Minor | `adapter/out/external/ProductLookupAdapter.java:107-122` vs `support/campaign/ScopeResolver.java:20-22` | ScopeResolver ghi "tra một lần cho mỗi loại", nhưng `lookupCollectionLabelsByIds` gọi `fetchCollection` **tuần tự, mỗi bộ sưu tập một lần**, tức N+1 call với timeout 60s mỗi lần. Ở chỗ này chiến dịch gắn nhiều bộ sưu tập vẫn sinh ra N lời gọi. | Dùng endpoint lô của pp-product nếu có. Nếu không thì sửa javadoc và đặt giới hạn số collection / ngân sách thời gian. |
| 🟡 Minor | `CampaignDetailAggregationService.java:162` | Chỉ lùi về `campaign.validationRuleId()` khi `bindings` rỗng. Nếu binding đầu tiên có `ruleId` null/blank thì trả `null` dù chiến dịch có rule id. Các binding sau `get(0)` bị bỏ qua. | Viết `ruleId = pick(firstNonBlank(binding.ruleId), campaign.validationRuleId())`. |
| 🟡 Minor | `support/campaign/ValiditySectionMapper.java:100-103` | Khi lùi về bản chụp pp-campaign, `ValidityHoursPerDay.dayOfWeek` bị bỏ (truyền `null` cho `daysOfWeek`). CSKH thấy khung giờ mà không biết áp cho thứ nào. Nguồn có dữ liệu này. | Map `dayOfWeek` (String) sang `List<Integer>` theo quy ước "Thứ Hai = 1". |
| 🟡 Minor | `ValiditySectionMapper.java:70-88` | Binding chỉ khai 1 mốc vẫn thắng cả 2 mốc, nên mốc còn lại thành null, xoá mất mốc đúng của chiến dịch. Đây chính là điều javadoc dòng 19-21 nói muốn tránh. | Chọn từng mốc riêng: `pick(binding.validFrom(), campaignStart)`, tương tự cho mốc kết thúc. |
| 🟡 Minor | `support/campaign/CouponConfigSectionMapper.java:80-86` | `maxUsesPerCode` lùi về `maxUsesPerCustomer`, rồi về `codeConfig.usageLimitPerCustomer`, tức trộn ngữ nghĩa "theo mã" với "theo khách". Cùng một giá trị có thể hiện ở cả 2 ô `maxUsesPerCode` và `maxUsesPerCustomer`. | Nếu đây là quirk của CMS cần giữ thì ghi nguồn quyết định (ticket). Nếu không thì tách bạch 2 trường. |
| 🟡 Minor | `adapter/config/BffAsyncConfig.java:3-4` | Import trùng `brave.propagation.CurrentTraceContext` (Sonar S1128), do commit này đưa vào. Field `currentTraceContext` (dòng 34) cũng bị dời lên trên khối hằng số, trông như sót lại khi resolve conflict. | Xoá dòng import trùng, trả field về vị trí cũ. |
| 🟡 Minor | `support/campaign/DiscountSectionMapper.java:75-83`, `ValiditySectionMapper.java:32-39` | Javadoc bị lặp 2 khối liền nhau (bản cũ không xoá), tạo ra Javadoc mồ côi. | Xoá khối thừa. |
| 🟡 Minor | `port/out/CampaignDetailViewPort.java:3`, `CouponConfigViewPort.java:3`, `DiscountFormulaViewPort.java:3`, `ValidationRuleViewPort.java:3-4`, `CampaignDetailAggregationService.java:6-9`, 6 mapper `support/campaign/*` | Tầng application (port out, service, mapper) import DTO của `adapter.out.external.dto`, khiến hướng phụ thuộc hexagonal bị đảo. Repo đã có tiền lệ nhưng commit này nhân rộng mạnh (khoảng 12 file). | Ngắn hạn thì chấp nhận được. Về lâu dài nên đưa model "view" vào `application` và để adapter map sang. |
| 🟡 Minor | Test | Không có test cho 4 adapter mới (404 thành `CampaignNotFoundException`, 5xx/timeout thành `DownstreamUnavailableException`, thân rỗng). Không có test web layer cho 400 (regex UUID) / 404 / 502. Service test dùng executor `Runnable::run`, nên không phủ được đồng thời / pool đầy / timeout. | Bổ sung test adapter và `@WebMvcTest` cho các mã lỗi. |
| 🔵 Info | `CampaignDetailFeignClient.java:12-14`, `CouponConfigFeignClient.java:15-17` | Javadoc lớp vẫn ghi "chỉ dùng cho backfill nội bộ, luồng đọc không gọi ra ngoài". Không còn đúng từ commit này. | Cập nhật javadoc. |
| 🔵 Info | `ProductLookupAdapter.java:51` | Đổi `log.warn` thành `log.error` cho đường SKU lookup có sẵn của app khách, vốn là best-effort. Thay đổi ngoài phạm vi commit, có thể làm ồn alert. | Tách ra commit riêng hoặc giữ `warn`. |
| 🔵 Info | `support/campaign/DiscountSectionMapper.java:84-106` | Nhánh dự phòng để `currency = null` trong khi `budget.currencyCode` có sẵn. Số tiền được trả dạng JSON number (BigDecimal). Cần thống nhất với quy ước toàn hệ thống để tránh JS làm tròn. | Cân nhắc lùi về `budget.currencyCode` khi khớp nghiệp vụ. |
| 🔵 Info | Mọi file mới | Không có header `// id: ... \| dd_version: ...` (cả repo cũng chưa có). | Như ghi chú ở commit trên. |

**Ghi chú:**
- Thiết kế rõ ràng: chia 3 đợt, tác vụ chỉ nộp từ luồng request nên tránh được deadlock pool lồng nhau. Có bể luồng riêng, không ảnh hưởng luồng app khách hàng. Context (MDC/trace/security) được propagate qua `TaskDecorator`.
- Nguồn chính map lỗi đúng: 404 thành `CampaignNotFoundException`, lỗi khác thành `DownstreamUnavailableException` (502). Không gọi nguồn phụ khi chiến dịch không tồn tại (có test).
- Tiền dùng `BigDecimal`, thời gian dùng `Instant`/`OffsetDateTime`. Không có `@Transactional` bao lời gọi ngoài. Input được validate bằng `@NotBlank @Size @Pattern(UUID)` + `@Validated`. Log chỉ ghi campaignId, không ghi PII.
- `ConditionsSectionMapper` chống vòng tham chiếu (có test). `NullSafe` giúp giữ cognitive complexity thấp. Không class main nào vượt 400 dòng (lớn nhất là `CustomerCareApi` 301 dòng). Sonar S125/S1192/S3776 không vi phạm, S1128 vi phạm 1 chỗ (import trùng ở trên).
- Đưa chuỗi mô tả 401/403/page/size ra hằng số (S1192) và `copyIfPresent` trong controller là refactor hợp lý.

---

