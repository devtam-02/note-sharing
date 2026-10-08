# Read model của vtm-bff: trước và sau phần việc CSKH

Mốc so sánh: commit `f66471f` (`fix(bff-vtm): chien dich bat Giu hieu luc thi ma co han rieng cung
theo thoi luong (PROM-1569)`) → `1dfffcc`. Nhánh `tamntt25/cskh-customer-care-api`.

Trong khoảng này còn vài commit **không liên quan read model** của người khác (feature flag
Unleash, preview ảnh MinIO, log chẩn đoán Feign) — tài liệu này bỏ qua chúng.

9 commit thuộc phần CSKH:

```
55e7638  khai API spec /api/cskh/v1
04991c4  schema read model cho CSKH (changeset 022, 023)
942d537  backfill campaign_code, coupon_type
a152843  nối live-path cho campaign_code, coupon_type
9118f14  trạng thái gán mã + tồn kho mã
235bd46  backfill trạng thái gán mã cho dòng cũ
bc0bd98  lịch sử tác động + trạng thái sử dụng (redemption + cashback)
aeb64ad  API tra cứu ưu đãi
1dfffcc  bản ghi giả lập cho chiến dịch khách chưa có mã
```

---

## 1. Tóm tắt một bảng

| | Trước (`f66471f`) | Sau (`1dfffcc`) |
|---|---|---|
| Bảng của read model | 4 | **7** (+`offer_audit`, +`campaign_stock`, +`cashback_participation`) |
| View `customer_voucher_view` | 53 cột | **61 cột** |
| Cột thêm vào bảng cũ | — | **8** (6 ở `voucher_ownership`, 1 ở `campaign_offer`, 1 ở `coupon_display`) |
| Consumer ghi read model | 6 | **8** (+`RedemptionAuditConsumer`, +`CashbackAuditConsumer`) |
| Topic đang nghe | 6 | **8** (+`promotion_redemption_event`, +`promotion_cashback_event`) |
| Endpoint backfill nội bộ | 7 | **11** |
| API đọc phục vụ CSKH | 0 | **2/3** (`/api-specs`, `/customer-offers`; `/customer-offers/audit` còn 501) |
| Port lookup REST sang service nguồn | 6 | **7** (+`CampaignStockLookupPort`) |

---

## 2. Thay đổi schema

### 2.1 Cột thêm vào bảng có sẵn — changeset `022`

| Bảng | Cột mới | Dùng cho cột nào của màn hình CSKH |
|---|---|---|
| `voucher_ownership` | `assignment_status` | Trạng thái gán mã |
| | `assigned_at` | Ngày gán — **tách khỏi `received_at`** để không đổi nghĩa cột đang phục vụ app khách hàng |
| | `usage_status` | Trạng thái sử dụng |
| | `last_session_id` | Mã phiên / giao dịch |
| | `last_session_at` | Ngày sử dụng — **khác `redeemed_at`**: một phiên có thể tạo rồi huỷ mà không bao giờ đổi thành công |
| | `last_event_at` | Mốc chặn event tới muộn |
| `campaign_offer` | `campaign_code` | Mã chiến dịch |
| `coupon_display` | `coupon_type` | Loại chiến dịch |

Cùng changeset: thêm index `idx_ownership_customer_campaign (customer_source_id, campaign_id)` và
dựng lại view với 8 cột trên.

### 2.2 Bảng mới

| Bảng | Changeset | Vì sao cần |
|---|---|---|
| `offer_audit` | `023` | Nguồn duy nhất cho lịch sử tác động. Chỉ thêm, không sửa — event tới muộn vẫn để lại dấu vết |
| `campaign_stock` | `023` | `voucher_ownership` chỉ có dòng khi mã **đã gán**; không có bảng này thì không phân biệt được "Chưa gán" với "Chưa sinh mã" |
| `cashback_participation` | `024` | Chiến dịch Hoàn tiền không phát mã nên không có dòng nào ở `voucher_ownership` |

---

## 3. Toàn bộ consumer đang ghi read model

8 consumer, tất cả dùng chung group `${app.kafka.groups.bff-projection}`, `AckMode` thủ công,
và **cô lập lỗi theo từng event** — một event hỏng không được làm các event sau nó trong lô bị bỏ
qua trong khi offset vẫn commit.

| # | Consumer | Topic | Loại event | Ghi vào bảng | Trạng thái |
|---|---|---|---|---|---|
| 1 | `VoucherProjectionConsumer` | `promotion_customer_event` | `CustomerVoucherEvent` (Added / Revoked / Redeemed) | `voucher_ownership`, **`offer_audit`** | cũ, **có sửa** |
| 2 | `VoucherStatusProjectionConsumer` | `promotion_voucher_event` | `VoucherEvent` | `voucher_ownership` | cũ, không đổi |
| 3 | `CampaignProjectionConsumer` | `promotion_campaign_event` | `CampaignEvent` | `campaign_offer` | cũ, **có sửa** |
| 4 | `CouponDisplayProjectionConsumer` | `promotion_coupon_config_event` | `CouponConfigEvent` | `coupon_display`, **`campaign_stock`** | cũ, **có sửa** |
| 5 | `OfferDiscountProjectionConsumer` | `promotion_discount_event` | `DiscountEvent` | `campaign_offer` | cũ, không đổi |
| 6 | `ValidationRuleProjectionConsumer` | `promotion_validation_event` | `ValidationEvent` | `validation_rule` | cũ, không đổi |
| 7 | **`RedemptionAuditConsumer`** | `promotion_redemption_event` | `RedemptionResultEvent` | `offer_audit`, `voucher_ownership` | **mới** |
| 8 | **`CashbackAuditConsumer`** | `promotion_cashback_event` | `CashbackSuccessEvent`, `CashbackFailedEvent` | `offer_audit`, `cashback_participation` | **mới** |

### 3.1 Ba consumer cũ bị sửa những gì

- **`VoucherProjectionConsumer`** — ngoài việc dựng dòng sở hữu như cũ, nay ghi thêm một dòng
  `offer_audit` hành động `ASSIGN`, và đặt `assignment_status` + `assigned_at`. Thu hồi mã thì
  đặt `assignment_status = revoked`, giữ dòng lại để tra cứu lịch sử.
- **`CampaignProjectionConsumer`** — sau khi chiếu xong, gọi lấp `campaign_code` bằng lookup REST
  sang pp-campaign. Lỗi lookup được nuốt riêng trong `try` của nó, không làm hỏng projection.
- **`CouponDisplayProjectionConsumer`** — tương tự, lấp `coupon_type` và làm mới `campaign_stock`.

### 3.2 Hai consumer mới làm gì

Cả hai theo cùng một khung: **ghi lịch sử trước, đụng trạng thái sau**.

```
ghi offer_audit  (LUÔN LUÔN — bảng chỉ thêm)
so occurredAt với last_event_at đang lưu
  mới hơn  → cập nhật trạng thái sử dụng
  đến trễ  → KHÔNG ghi đè
ack offset
```

Chống trùng đặt ở adapter theo bộ ba `(event_id, voucher_code, action)` — đủ để một event tác
động nhiều mã vẫn ghi nhiều dòng, mà phát lại thì không nhân đôi lịch sử.

### 3.3 Ba chỗ payload không đủ, phải suy từ read model

| Thiếu gì | Ở event nào | Cách lấy |
|---|---|---|
| Số thuê bao | `RedemptionResultEvent` (chỉ có `customerId`) | tra `voucher_ownership` theo mã |
| Mã chiến dịch | `RedemptionResultEvent` (`campaignId` **luôn null**) | tra `voucher_ownership` theo mã |
| Số thuê bao | `CashbackFailedEvent` | tra các lần tham gia trước trong `cashback_participation` |

---

## 4. API mới

### 4.1 Phục vụ hệ thống CSKH — `/api/cskh/v1`

| Endpoint | Trạng thái | Ghi chú |
|---|---|---|
| `GET /api-specs` | ✅ | Trả đặc tả OpenAPI rút gọn của riêng thao tác tra cứu, để FE lấy danh mục dropdown từ `enum` + `x-enum-labels` |
| `GET /customer-offers` | ✅ | Tra cứu ưu đãi, hợp nhất 3 nhánh bằng `UNION ALL` ở tầng SQL |
| `GET /customer-offers/audit` | ❌ 501 | Dữ liệu `offer_audit` đã đủ, chưa viết phần xử lý |

Tiền tố `/api/cskh/v1` **cố ý không chứa** `/api/v1/` — đó là điều kiện lọc của `VtmAuthFilter`.
Nhờ vậy filter xác thực khách hàng không chạm vùng này. Đổi tiền tố thành dạng có `/api/v1/` sẽ
khiến token khách hàng bất kỳ đi qua được các endpoint tra cứu dữ liệu người khác.

**Ba nhánh của `/customer-offers`:**

1. Ưu đãi dạng mã khách đang giữ — đọc `customer_voucher_view`.
2. Ưu đãi hoàn tiền khách đã tham gia — đọc `cashback_participation`.
3. Chiến dịch đang chạy khách **chưa** có mã — dựng từ `campaign_stock`, cho ra hai trạng thái
   `unpublished` và `not_generated`.

### 4.2 Backfill nội bộ — thêm 4 endpoint

| Endpoint | Lấp gì | Nguồn |
|---|---|---|
| `POST /internal/backfill/campaign-code` | `campaign_offer.campaign_code` | REST pp-campaign |
| `POST /internal/backfill/coupon-type` | `coupon_display.coupon_type` | REST pp-coupon |
| `POST /internal/backfill/campaign-stock` | cả 4 cột của `campaign_stock` | REST pp-coupon |
| `POST /internal/backfill/assignment-status` | `assignment_status`, `assigned_at` cho dòng có từ trước | suy từ chính dòng sở hữu |

7 endpoint cũ (`voucher-expiry`, `applicable-products`, `campaign-created-at`, `campaign-validity`,
`coupon-display-campaign-id`, `coupon-display-from-catalog`, `campaign-offer-discount`) giữ nguyên.

---

## 5. Lookup REST thêm mới

Thêm một port (`CampaignStockLookupPort`) và mở rộng `CouponConfigLookupPort` bằng
`findConfigByCampaign` — gộp ba lượt tra thành một lượt gọi.

Tồn kho mã **cố ý không đếm tăng giảm theo event**: đếm kiểu đó trôi số ngay khi lỡ một event,
còn hỏi thẳng pp-coupon thì luôn ra con số thật tại thời điểm hỏi.

| Cần gì | Gọi đâu | Đọc gì |
|---|---|---|
| Số mã chưa gán | `GET /api/v1/vouchers?campaignId=X&distributionStatus=UNPUBLISHED&size=1` | `totalElements` |
| Tổng số mã đã sinh | `GET /api/v1/vouchers?campaignId=X&size=1` | `totalElements` |
| Giá trị mã chung | `GET /api/v1/vouchers?couponConfigId=X&size=1` | `code` |
| Loại mã + cờ sinh tự động | `GET /api/v1/coupon/configs/campaign/{campaignId}` | `couponType`, `autoReplenish` |

---

## 6. Hai lỗi có sẵn, lộ ra khi chạy thật

Cả hai chỉ lộ khi ứng dụng khởi động được tới bước `ddl-auto=validate`, đã sửa trong `9118f14`:

1. `CampaignOfferEntity.campaignCode` bị chèn vào **giữa** `@Column(name="campaign_type")` và field
   `campaignType` → hai field trỏ cùng một cột vật lý, `entityManagerFactory` không dựng được.
2. `customer_voucher_view.applies_to_all` ra kiểu `int` chứ không phải `tinyint(1)` vì đi qua
   `COALESCE` (bắt buộc do `campaign_offer` nối LEFT JOIN). Lỗi có từ changeset `020`, trước đây
   bị che vì view trên dev là BASE TABLE.

Một sai cấu hình nữa, ngoài phạm vi commit: `app.kafka.topics.customer-event` trên dev trỏ topic
`dit` thay vì `promotion_customer_event` → consumer số 1 không nhận được event nào. Đây nhiều khả
năng là lý do read model dev rỗng từ đầu.

---

## 7. Những gì KHÔNG đổi

- Không bỏ hay đổi nghĩa cột nào đang có. 53 cột cũ của view giữ nguyên tên và ngữ nghĩa.
- Không đụng tới 6 consumer cũ ở phần chiếu dữ liệu cho app khách hàng — ba consumer bị sửa chỉ
  được **thêm** việc, trong `try` riêng, lỗi không lan sang phần chiếu cũ.
- API #12, #13, #01 của app khách hàng giữ nguyên hợp đồng. Đối soát sau khi sửa: API #13 khớp
  tuyệt đối với read model ở 6 thuê bao đông voucher nhất; API #12 khớp 10/10 trường trên mẫu
  kiểm; độ trễ chiếu event 0,83-1,14 giây, không xấu đi.
- Không thêm phụ thuộc runtime nào mới ngoài hai topic Kafka.

---

## 8. Còn lệch so với đặc tả

| Chỗ lệch | Vì sao | Ảnh hưởng |
|---|---|---|
| Bản ghi giả lập chưa lọc theo segment | Read model không có dữ liệu segment | Trả mọi chiến dịch đang chạy (568 trên dev) thay vì riêng chiến dịch khách đủ điều kiện |
| Tên chiến dịch lấy `coupon_display.title` | Read model không lưu tên chiến dịch | Thực chất là tên voucher; chiến dịch hoàn tiền lùi về `campaign_code` |
| Lọc theo tên chưa bỏ dấu | Cột `title_normalized` mới chỉ hạ chữ thường | Gõ có dấu khác không dấu ra kết quả khác nhau |
| `usage_status = pending` suy từ `status = RESERVED` | pp-redemption không phát sự kiện mở phiên | Chấp nhận được, nhưng không phải nguồn trực tiếp |
| `/api/cskh/v1` chưa có xác thực | Chưa làm theo thống nhất | Phải có trước khi phục vụ dữ liệu thật |
