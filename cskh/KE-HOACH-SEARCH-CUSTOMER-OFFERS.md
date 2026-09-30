# Kế hoạch phát triển tính năng searchCustomerOffers

API: `GET /promotion/promotion-vtm-bff/api/cskh/v1/customer-offers`

Nguồn yêu cầu: `PRM_KBNV_CSKH_001_Tích hợp hệ thống CSKH`, mục 2.3 và 4.4

Ngày lập: 2026-09-30 · Phạm vi: `p2_promotion-vtm-bff` · Đặc tả đã có ở `CustomerServiceApi`

---

## 1. Đính chính tài liệu phân tích trước

Khảo sát thêm mã nguồn các service ở `../promotion` và `p2_schema-service` đã gỡ được phần lớn
điểm chặn, đồng thời cho thấy **hai kết luận trước là sai**.

| # | Kết luận cũ | Thực tế | Bằng chứng |
|---|---|---|---|
| 1 | `offerType` chưa có nguồn, cần thêm cột `campaign_offer.offer_type` | **Không cần cột mới.** Suy được từ `campaign_offer.campaign_type` đã có sẵn: `DISCOUNT_COUPON` là Mã giảm giá, `CASHBACK` là Hoàn tiền | `p2_promotion-campaign/.../domain/enums/CampaignType.java` gồm DISCOUNT_COUPON, PROMOTION, CASHBACK, GIFT, REFERRAL, LOYALTY |
| 2 | "Loại chiến dịch" (Mã tạo hàng loạt / Mã chung) đã có ở `campaign_offer.campaign_type` | **Sai trục dữ liệu.** Đây là `CouponType` của pp-coupon (BULK / SHARED), nằm ở coupon config chứ không phải campaign | `p2_promotion-coupon/.../domain/enums/CouponType.java` |
| 3 | Chưa rõ chiến dịch Hoàn tiền có đi qua `promotion_campaign_event` không | **Có.** `campaign_type` nằm trong payload nên chiến dịch CASHBACK vẫn vào `campaign_offer` | `CampaignStatusChangedEventPayload` có `campaign_type` |
| 4 | Không tìm thấy service cashback | **Có tại `../promotion/promotion-cashback`**, kèm bộ event đầy đủ trong `p2_schema-service/.../cashback/event/` | `CashbackEvent`, `CashbackResultEvent`, `CashbackRevertedEvent`... |
| 5 | Mã chiến dịch không có nguồn | **Có `Campaign.code` ở pp-campaign**, nhưng KHÔNG nằm trong payload của bất kỳ campaign event nào | `Campaign.java:31`, `CampaignEntity.java:38`; `CampaignStatusChangedEventPayload` không có trường code |

**Hệ quả cho DTO đã viết**: `CustomerOfferItem.campaignType` hiện khai `allowableValues = {"BULK_CODE", "SHARED_CODE"}`
— cần đổi sang vocabulary thật của pp-coupon (`BULK`, `SHARED`) hoặc chốt lại quy ước dịch.

---

## 2. Nguồn dữ liệu đã chốt

### 2.1 Tám tham số lọc

| Filter | Cột đối chiếu | Trạng thái |
|---|---|---|
| `msisdn` (bắt buộc, khớp chính xác) | `voucher_ownership.customer_source_id` | Có sẵn |
| `voucherCode` (contains) | `voucher_ownership.voucher_code` | Có sẵn |
| `campaignName` (contains, bỏ dấu) | `coupon_display.title_normalized` | Có sẵn |
| `campaignStatus` | `campaign_offer.campaign_status` | Có sẵn, cần bảng ánh xạ vocabulary |
| `offerType` | `campaign_offer.campaign_type` | Có sẵn, cần bảng ánh xạ |
| `productCode` | `campaign_offer.applicable_products` | Có sẵn |
| `assignmentStatus` | cột mới `voucher_ownership.assignment_status` | Cần thêm |
| `usageStatus` | cột mới `voucher_ownership.usage_status` | Cần thêm |

### 2.2 Mười ba cột kết quả

| Cột UI | Nguồn | Trạng thái |
|---|---|---|
| Tên chiến dịch | `coupon_display.title` | Có sẵn |
| Mã chiến dịch | `Campaign.code` của pp-campaign, **không có trong event** | Cần lookup REST + backfill |
| Trạng thái chiến dịch | `campaign_offer.campaign_status` | Có sẵn |
| Thời gian chiến dịch | `campaign_offer.start_date`, `.expiration_date` | Có sẵn |
| Loại ưu đãi | suy từ `campaign_offer.campaign_type` | Có sẵn, chỉ cần ánh xạ |
| Loại chiến dịch | `CouponType` của pp-coupon, **không có trong `CouponConfigCreatedEvent`** | Cần lookup REST + backfill |
| SP áp dụng | `campaign_offer.applicable_products` | Có sẵn |
| Mã ưu đãi | `voucher_ownership.voucher_code`, kèm nhãn thay thế | Có sẵn + logic mới |
| Trạng thái gán mã | suy từ bản ghi + tồn kho mã của chiến dịch | Cần cột mới + nguồn tồn kho |
| Ngày gán | `voucher_ownership.received_at` | Có sẵn, cần xác nhận ngữ nghĩa |
| Trạng thái sử dụng | pp-redemption và pp-cashback | Cần consumer mới |
| Ngày sử dụng | ngày tạo session gần nhất, khác `redeemed_at` | Cần cột mới |
| Thao tác | FE tự dựng từ `hasHistory` | Có sẵn |

### 2.3 Mẫu sẵn có để lấp trường thiếu

Hai trường `Mã chiến dịch` và `Loại chiến dịch` không nằm trong event. Codebase đã có **mẫu xử lý
đúng tình huống này**: `CampaignCreatedAtBackfillService` dùng `CampaignDetailLookupPort` gọi REST
sang pp-campaign để lấp trường event không mang, kèm endpoint `/internal/backfill/*` để chạy lại
cho dữ liệu cũ. Bám theo mẫu này thay vì mở rộng event của service khác — ít phụ thuộc đội ngoài,
triển khai được ngay.

---

## 3. Sáu đầu việc

Thứ tự là thứ tự phụ thuộc. Đầu việc 1 và 2 chạy song song được.

### Đầu việc 1 — Chuẩn hoá vocabulary và bảng ánh xạ

Nền cho mọi phần còn lại: mọi nơi khác đều tiêu thụ kết quả của đầu việc này.

**Việc cụ thể**

- Viết lớp ánh xạ hai chiều giữa giá trị UI và enum nguồn:

| Trục | Giá trị UI | Enum nguồn |
|---|---|---|
| `offerType` | `voucher` | `CampaignType.DISCOUNT_COUPON` |
| | `cashback` | `CampaignType.CASHBACK` |
| `campaignStatus` | `initializing`, `active`, `running`, `paused`, `expired`, `error` | enum của pp-campaign |
| Loại chiến dịch | `BULK`, `SHARED` | `CouponType` |

- **Quyết định cần chốt**: read model lưu nguyên văn enum pp-campaign và từ code thấy có
  `RUNNING`, `PAUSED`, `DISABLING`, `DELETING`, `DELETED`, `EXPIRED`, `ERROR`; trong khi danh mục UI
  có `initializing`, `active` (chưa thấy trong code) và **không có** ba trạng thái xoá. Phải chốt:
  ba trạng thái đó gom về đâu, hay chiến dịch ở trạng thái đó bị loại khỏi kết quả tra cứu.
- Giá trị `all` ở mọi filter nghĩa là không áp điều kiện.
- `CampaignType` còn `PROMOTION`, `GIFT`, `REFERRAL`, `LOYALTY` — chốt cách xử lý khi gặp
  (loại khỏi kết quả, hay trả về với `offerType` rỗng).

**Tiêu chí hoàn thành**: test tham số hoá phủ mọi giá trị enum nguồn, kể cả giá trị không có trong
danh mục UI, không ném lỗi.

---

### Đầu việc 2 — Bổ sung trường chiến dịch còn thiếu

**Việc cụ thể**

- Liquibase: thêm `campaign_offer.campaign_code`, `coupon_display.coupon_type`; cập nhật view
  `customer_voucher_view`. Theo mẫu changeset `006-replace-view-customer-voucher.xml`,
  đánh số tiếp sau `021`.
- Mở rộng `CampaignDetailLookupPort` lấy thêm `code`; ghi vào `campaign_offer` trong
  `CampaignProjectionConsumer`.
- Thêm port lookup coupon config của pp-coupon lấy `couponType`; ghi vào `coupon_display` trong
  `CouponDisplayProjectionConsumer`.
- Hai endpoint backfill mới dưới `/internal/backfill` cho dữ liệu đã tồn tại, theo đúng mẫu
  `campaign-created-at`.

**Tiêu chí hoàn thành**: liquibase chạy sạch và rollback được; backfill điền đủ cho dữ liệu cũ;
bản ghi mới có `campaign_code` và `coupon_type` ngay khi event tới.

---

### Đầu việc 3 — Trạng thái gán mã và tồn kho mã

**Việc cụ thể**

- Thêm cột `voucher_ownership.assignment_status`, `assigned_at`.
  `assigned_at` lấy từ `received_at` đang có — **cần xác nhận `received_at` đúng là thời điểm gán**.
- Bảng mới `cskh_campaign_stock` phục vụ phân biệt "Chưa gán" với "Chưa sinh mã" và logic bản ghi
  giả lập ở đầu việc 5:

```
cskh_campaign_stock
  campaign_id            PK
  total_code_count       int      -- CouponConfigCreatedEvent.voucherCount
  unassigned_code_count  int      -- CAN NGUON MOI
  auto_generate          boolean  -- CAN NGUON MOI
  shared_code            varchar  -- CAN NGUON MOI
  updated_at             datetime
```

- `CouponConfigCreatedEvent` **đã có `voucherCount`** (tổng số mã). Ba trường còn lại chưa có nguồn:
  hoặc pp-coupon bổ sung event tồn kho, hoặc dùng lookup REST theo mẫu đầu việc 2.

**Tiêu chí hoàn thành**: ba giá trị `published` / `unpublished` / `not_generated` phân biệt đúng
trên dữ liệu thật.

---

### Đầu việc 4 — Trạng thái sử dụng

Phần nặng nhất, và là phần duy nhất bắt buộc phải có consumer mới.

**Việc cụ thể**

- Thêm cột `voucher_ownership.usage_status`, `last_session_id`, `last_session_at`, `last_event_at`.
  **`last_session_at` khác `redeemed_at` đang có**: SRS yêu cầu ngày tạo phiên gần nhất, không phải
  ngày đổi thành công — một phiên có thể tạo rồi huỷ mà không bao giờ redeem.
- Bảng `processed_event` cho idempotency (yêu cầu mục 4.4).
- `RedemptionAuditConsumer` nghe event của pp-redemption; ánh xạ:

| Điều kiện | `usage_status` |
|---|---|
| Không có phiên đổi thưởng | `unused` |
| Trạng thái giữ chỗ Active | `pending` |
| Redeemed | `success` |
| Released hoặc Expired | `failed` |

- `CashbackAuditConsumer` nghe event của pp-cashback tại `../promotion/promotion-cashback`;
  ánh xạ: có giao dịch hoàn tiền là `participated`, không có là `not_participated`.
- Logic event đến trễ theo mục 4.4: luôn ghi nhận, **chỉ cập nhật trạng thái khi event mới hơn**
  `last_event_at`.

**Tiêu chí hoàn thành**: test đơn vị cho 3 ca — event mới, event trùng, event đến trễ; phủ cả hai
nhánh Mã giảm giá và Hoàn tiền.

---

### Đầu việc 5 — Truy vấn, sắp xếp, phân trang và bản ghi giả lập

**Việc cụ thể**

- Mở rộng `VoucherSearchSpecifications` cho 8 điều kiện lọc; bỏ qua điều kiện khi giá trị là `all`.
- Sắp xếp theo SRS mục 2.3: ngày bắt đầu chiến dịch giảm dần, ngày gán mã giảm dần, ngày sử dụng
  giảm dần, mã chiến dịch tăng dần. Đặt trọn trong specification, truyền `Sort.unsorted()` cho
  `PageRequest` — đúng cách `CustomerVoucherViewAdapter.search()` đang làm, vì Spring Data ghi đè
  `orderBy` của specification khi `Pageable` có sort.
- **Bản ghi giả lập** (mục 4.4): khi chiến dịch đang chạy, khách thoả cấu hình segment và không có
  bản ghi nào, bổ sung một dòng:

| Tình huống | `voucherCode` | `hasHistory` |
|---|---|---|
| Còn mã đã sinh chưa gán | `"Còn mã"` | false |
| Hết mã chưa gán, có sinh mã tự động | `"-"` | false |
| Chiến dịch mã chung | giá trị mã chung | false |
| Chiến dịch Hoàn tiền | null, `usageStatus = not_participated` | false |

- **Điểm chưa có lời giải**: điều kiện "khách thoả cấu hình segment". Read model không lưu segment
  của khách. Ba hướng: bỏ điều kiện segment (sai SRS), gọi REST pp-segment lúc đọc (chậm, phá
  nguyên tắc không fan-out ở mục 4.1), hoặc chiếu segment membership vào read model (đúng kiến trúc,
  tốn công nhất). **Cần chốt trước khi làm đầu việc này.**

**Tiêu chí hoàn thành**: phân trang và tổng số đếm đúng khi có bản ghi giả lập; thứ tự khớp SRS.

---

### Đầu việc 6 — Ghép nối và bỏ 501

**Việc cụ thể**

- Thêm in-port `CustomerServiceUseCase`, service hiện thực, thay
  `ResponseStatusException(NOT_IMPLEMENTED)` trong `CustomerServiceController.searchCustomerOffers`.
- Sửa `CustomerOfferItem.campaignType` sang vocabulary thật (`BULK` / `SHARED`).
- Đầu việc 1 của kế hoạch tổng — filter xác thực nhân viên — **phải xong trước khi lên UAT**,
  nếu không đây là endpoint tra cứu dữ liệu khách hàng không cần xác thực.

**Tiêu chí hoàn thành**: gọi thật trả dữ liệu đúng 13 cột; Swagger hiển thị đủ dropdown; có test
tích hợp cho ít nhất một tổ hợp lọc của mỗi loại chiến dịch.

---

## 4. Phụ thuộc

| Đầu việc | Phụ thuộc | Có chặn bởi câu hỏi mở không |
|---|---|---|
| 1. Vocabulary | Không | Có — ba trạng thái xoá, bốn CampaignType còn lại |
| 2. Trường chiến dịch | Không | Không |
| 3. Gán mã và tồn kho | 2 | Có — nguồn tồn kho mã |
| 4. Trạng thái sử dụng | 1 | Không, pp-cashback đã có event |
| 5. Truy vấn | 1, 2, 3, 4 | Có — điều kiện segment |
| 6. Ghép nối | Tất cả | Không |

Thứ tự khuyến nghị: **1 và 2 song song → 3 và 4 song song → 5 → 6.**

---

## 5. Câu hỏi cần chốt

| # | Câu hỏi | Chặn |
|---|---|---|
| 1 | Ba trạng thái `DISABLING`, `DELETING`, `DELETED` của pp-campaign gom về giá trị UI nào, hay loại khỏi kết quả? | 1 |
| 2 | `CampaignType.PROMOTION`, `GIFT`, `REFERRAL`, `LOYALTY` xử lý thế nào khi gặp? | 1 |
| 3 | Số mã chưa gán, cờ sinh mã tự động và mã chung lấy từ đâu — pp-coupon bổ sung event, hay lookup REST? | 3, 5 |
| 4 | `voucher_ownership.received_at` có đúng là thời điểm gán mã cho khách không? | 3 |
| 5 | Điều kiện "khách thoả cấu hình segment" ở mục 4.4 đánh giá thế nào? | 5 |
| 6 | Giá trị UI của "Loại chiến dịch": giữ `BULK`/`SHARED` hay đặt vocabulary riêng? | 6 |

---

## 6. Rủi ro

| Rủi ro | Mức | Xử lý |
|---|---|---|
| Vùng `/api/cskh/v1` chưa có xác thực; khi bỏ 501 sẽ lộ dữ liệu khách hàng | **Cao** | Bắt buộc xong filter xác thực nhân viên trước khi lên UAT |
| Bản ghi giả lập làm sai tổng số đếm và phân trang | Trung bình | Thiết kế đếm riêng cho phần giả lập; test phân trang ở biên |
| Thêm cột vào view `customer_voucher_view` ảnh hưởng luồng app đang chạy | Trung bình | Chỉ thêm cột, không đổi hoặc bỏ cột cũ; chạy hồi quy API #12 và #13 |
| Consumer mới làm chậm projection đang có | Thấp | Consumer group riêng cho nhóm CSKH |
| Lookup REST lúc backfill gặp 403 như `ProductLookupAdapter` trên production | Trung bình | Kiểm quyền gọi liên service trước khi chạy backfill diện rộng |

---

## Phụ lục — Bằng chứng

```bash
# CampaignType: DISCOUNT_COUPON vs CASHBACK  -> offerType
cat /mnt/data/Data/code/staging_promotion/p2_promotion-campaign/src/main/java/vn/viettel/vds/promotion/campaign/domain/enums/CampaignType.java

# CouponType: BULK vs SHARED  -> Loai chien dich
cat /mnt/data/Data/code/staging_promotion/p2_promotion-coupon/src/main/java/vn/viettel/vds/promotion/coupon/domain/enums/CouponType.java

# Campaign.code co that nhung khong nam trong event payload
grep -n "private String code" /mnt/data/Data/code/staging_promotion/p2_promotion-campaign/src/main/java/vn/viettel/vds/promotion/campaign/domain/model/Campaign.java
grep -n "JsonProperty" /mnt/data/Data/code/staging_promotion/p2_schema-service/src/main/java/vn/viettel/vds/promotion/campaign/event/CampaignStatusChangedEventPayload.java

# CouponConfigCreatedEvent co voucherCount nhung khong co couponType
grep -nE "private " /mnt/data/Data/code/staging_promotion/p2_schema-service/src/main/java/vn/viettel/vds/promotion/couponconfig/event/CouponConfigCreatedEvent.java

# pp-cashback va bo event day du
ls /mnt/data/Data/code/promotion/promotion-cashback
ls /mnt/data/Data/code/staging_promotion/p2_schema-service/src/main/java/vn/viettel/vds/promotion/cashback/event/

# Mau backfill lay truong event khong mang
sed -n '1,40p' /mnt/data/Data/code/staging_promotion/p2_promotion-vtm-bff/src/main/java/vn/viettel/vds/promotion/bff/vtm/application/service/CampaignCreatedAtBackfillService.java
```
