# Read model: hiện tại và đề xuất

Phạm vi: `p2_promotion-vtm-bff`, MariaDB `promotion_vtm_bff`

Ngày lập: 2026-09-30 · Phục vụ: `searchCustomerOffers` và `getAuditTrail` của nhóm API CSKH

> Kế hoạch `KE-HOACH-SEARCH-CUSTOMER-OFFERS.md` có phần cập nhật read model nhưng nằm rải ở
> đầu việc 2, 3 và 4. Tài liệu này gom lại thành một bức tranh để dễ rà và dễ ước lượng.

---

## 1. Read model hiện tại

Bốn bảng nền, một view. View là thứ duy nhất tầng ứng dụng đọc.

```mermaid
erDiagram
    voucher_ownership {
        string id PK
        string customer_source_id "msisdn - khoa tra cuu"
        string voucher_id
        string voucher_code
        string customer_id
        string campaign_id FK
        string coupon_config_id FK
        string group_type
        string status "AVAILABLE RESERVED REDEEMED REVOKED EXPIRED"
        instant status_updated_at
        instant redeemed_at
        instant reserved_until
        string reserved_session_id
        int current_total_uses
        int max_uses_per_code
        instant start_date
        instant expiration_date
        instant received_at
        instant created_at
        instant updated_at
    }
    campaign_offer {
        string campaign_id PK
        string campaign_type "DISCOUNT_COUPON CASHBACK PROMOTION GIFT REFERRAL LOYALTY"
        string campaign_status
        instant status_updated_at
        string time_slot
        string validation_rule_id FK
        string applicable_service_codes
        string excluded_service_codes
        boolean applies_to_all
        string discount_type
        string discount_method
        decimal discount_value
        string discount_percentage
        string discount_label
        decimal min_order_value
        decimal max_discount_amount
        string included_products
        string excluded_products
        string applicable_products
        int max_total_uses
        int per_customer_limit
        instant start_date
        instant expiration_date
        instant created_at
        instant updated_at
        string metadata
        string campaign_description
    }
    coupon_display {
        string coupon_config_id PK
        string campaign_id
        string title
        string title_normalized "bo dau - phuc vu tim kiem"
        string merchant_name
        string merchant_name_normalized
        string description
        string logo_url
        string banner_url
        string usage_guide_url
        instant updated_at
        string metadata
        string tags
    }
    validation_rule {
        string rule_id PK
        string rule_code
        string description
        string rules
        decimal min_order_value
        decimal max_order_value
        instant updated_at
    }
    customer_voucher_view {
        string id PK "VIEW - khong phai bang"
    }

    voucher_ownership ||--o| campaign_offer : "campaign_id"
    voucher_ownership ||--o| coupon_display : "coupon_config_id"
    campaign_offer    ||--o| validation_rule : "validation_rule_id"
    customer_voucher_view }o--|| voucher_ownership : "LEFT JOIN 3 bang con lai"
```

### Vai trò của từng bảng

| Bảng | Giữ cái gì | Trả lời câu hỏi nào | Khoá chính |
|---|---|---|---|
| `voucher_ownership` | Quan hệ **một khách hàng sở hữu một mã** — và trạng thái hiện tại của quan hệ đó | "Khách này đang có những mã nào, mỗi mã đang ở trạng thái gì?" | `id` |
| `campaign_offer` | Thuộc tính **mức chiến dịch**: loại, trạng thái, thời gian hiệu lực, phạm vi sản phẩm, cấu hình giảm giá | "Chiến dịch này còn chạy không, áp cho sản phẩm nào, giảm bao nhiêu?" | `campaign_id` |
| `coupon_display` | Thông tin **hiển thị** của một coupon config: tiêu đề, tên đơn vị, mô tả, ảnh | "Hiện cái gì lên màn hình cho mã này?" | `coupon_config_id` |
| `validation_rule` | Luật kiểm tra nhanh khi áp mã: ngưỡng giá trị đơn tối thiểu và tối đa | "Đơn hàng có đủ điều kiện dùng mã không?" | `rule_id` |
| `customer_voucher_view` | **Không giữ gì** — là VIEW ghép 4 bảng trên | Là thứ duy nhất tầng ứng dụng đọc, để truy vấn không phải tự join | — |

Ba điểm đáng chú ý về cách phân chia:

**`voucher_ownership` là bảng sự kiện hoá, không phải danh mục voucher.** Mỗi dòng ứng với một lần
cấp mã cho một khách cụ thể. Chiến dịch chưa phát mã cho ai thì **không có dòng nào** — đây chính
là lý do nghiệp vụ CSKH cần thêm `campaign_stock` (xem mục 2.2).

**`campaign_offer` có hai đường ghi tách bạch** và cố ý không gộp: pp-campaign ghi nhóm meta,
pp-cashback ghi nhóm giảm giá qua `upsertDiscount`. Trộn hai nguồn vào một đường sẽ thành hai bên
cùng ghi đè một nhóm cột.

**Tách `coupon_display` khỏi `campaign_offer`** vì chúng khác khoá: một chiến dịch có thể có nhiều
coupon config. Gộp lại sẽ phải nhân bản thuộc tính chiến dịch cho mỗi config.

### Đường ghi dữ liệu hiện tại

| Bảng | Consumer | Topic | Service nguồn |
|---|---|---|---|
| `voucher_ownership` | `VoucherProjectionConsumer` | `customer-event` | pp-customer |
| | `VoucherStatusProjectionConsumer` | `promotion_voucher_event` | pp-coupon |
| `campaign_offer` | `CampaignProjectionConsumer` | `promotion_campaign_event` | pp-campaign |
| | `OfferDiscountProjectionConsumer` | `promotion_discount_event` | pp-cashback |
| `coupon_display` | `CouponDisplayProjectionConsumer` | `promotion_coupon_config_event` | pp-coupon-config |
| `validation_rule` | `ValidationRuleProjectionConsumer` | `promotion_validation_event` | pp-validation |

`campaign_offer` có **hai đường ghi tách bạch**: pp-campaign ghi nhóm meta, pp-cashback ghi nhóm
giảm giá. Ranh giới này là cố ý — xem chú thích trong `CampaignDetailDto`.

### View `customer_voucher_view`

Định nghĩa mới nhất ở changeset `020-validation-order-bounds.xml`, gồm **53 cột**, join:

```sql
FROM voucher_ownership o
LEFT JOIN campaign_offer  ofr ON ofr.campaign_id    = o.campaign_id
LEFT JOIN coupon_display  d   ON d.coupon_config_id = o.coupon_config_id
LEFT JOIN validation_rule vr  ON vr.rule_id         = ofr.validation_rule_id;
```

View đã bị định nghĩa lại nhiều lần (changeset 006, 007, 009, 010, 011, 012, 014, 015, 016, 018,
019, 020). Mỗi lần thêm cột đều phải `DROP` rồi `CREATE` lại toàn bộ.

### Hạn chế với nghiệp vụ CSKH

| Thiếu gì | Hệ quả |
|---|---|
| Không có bảng lịch sử | `getAuditTrail` không có nguồn |
| Không có trạng thái gán mã, trạng thái sử dụng | 2 trong 13 cột và 2 trong 8 filter của `searchCustomerOffers` không làm được |
| Không có mã chiến dịch, loại chiến dịch (BULK/SHARED) | 2 cột nữa không làm được |
| Không có tồn kho mã | Không phân biệt được "Chưa gán" với "Chưa sinh mã"; không làm được bản ghi giả lập |
| Không có bảng idempotency | Mục 4.4 yêu cầu chống xử lý trùng và xử lý event đến trễ |
| Chỉ xoay quanh voucher đã gán | Chiến dịch Hoàn tiền không có bản ghi nào |

---

## 2. Read model đề xuất

Giữ nguyên bốn bảng nền và view, **thêm 8 cột** và **3 bảng mới**. Không đổi hay bỏ cột nào đang có.

```mermaid
erDiagram
    voucher_ownership {
        string id PK
        string customer_source_id
        string campaign_id FK
        string coupon_config_id FK
        string status "giu nguyen"
        instant received_at "giu nguyen"
        instant redeemed_at "giu nguyen - ngay doi thanh cong"
        string assignment_status "MOI - published unpublished not_generated"
        instant assigned_at "MOI - tu received_at"
        string usage_status "MOI - unused pending success failed"
        string last_session_id "MOI"
        instant last_session_at "MOI - ngay tao phien gan nhat"
        instant last_event_at "MOI - moc so sanh event den tre"
    }
    campaign_offer {
        string campaign_id PK
        string campaign_type "giu nguyen - suy ra offerType"
        string campaign_status "giu nguyen"
        string applicable_products "giu nguyen"
        string campaign_code "MOI - lookup REST pp-campaign"
    }
    coupon_display {
        string coupon_config_id PK
        string title_normalized "giu nguyen"
        string coupon_type "MOI - BULK SHARED - lookup REST pp-coupon"
    }
    validation_rule {
        string rule_id PK
    }
    campaign_stock {
        string campaign_id PK "BANG MOI"
        int total_code_count "tu CouponConfigCreatedEvent.voucherCount"
        int unassigned_code_count "CAN NGUON"
        boolean auto_generate "CAN NGUON"
        string shared_code "CAN NGUON"
        instant updated_at
    }
    offer_audit {
        string id PK "BANG MOI"
        string customer_source_id
        string campaign_id
        string voucher_code
        instant occurred_at
        string action
        string product_scope
        string session_or_txn_id
        string result "SUCCESS FAILED"
        string note
        string event_id
        instant created_at
    }
    customer_voucher_view {
        string id PK "VIEW - them 8 cot moi"
    }

    voucher_ownership ||--o| campaign_offer : "campaign_id"
    voucher_ownership ||--o| coupon_display : "coupon_config_id"
    campaign_offer    ||--o| validation_rule : "validation_rule_id"
    campaign_offer    ||--o| campaign_stock : "campaign_id"
    offer_audit  }o--|| campaign_offer : "campaign_id"
    customer_voucher_view }o--|| voucher_ownership : "LEFT JOIN"
```

### 2.1 Cột bổ sung

| Bảng | Cột | Kiểu | Nguồn | Phục vụ |
|---|---|---|---|---|
| `voucher_ownership` | `assignment_status` | varchar(32) | suy từ bản ghi + `campaign_stock` | Cột và filter Trạng thái gán mã |
| | `assigned_at` | datetime(6) | `received_at` đang có | Cột Ngày gán |
| | `usage_status` | varchar(32) | consumer redemption và cashback | Cột và filter Trạng thái sử dụng |
| | `last_session_id` | varchar(64) | consumer redemption | Cột Mã phiên ở audit |
| | `last_session_at` | datetime(6) | consumer redemption | Cột Ngày sử dụng |
| | `last_event_at` | datetime(6) | `occurredAt` của event | So sánh event đến trễ |
| `campaign_offer` | `campaign_code` | varchar(64) | lookup REST pp-campaign | Cột Mã chiến dịch |
| `coupon_display` | `coupon_type` | varchar(32) | lookup REST pp-coupon | Cột Loại chiến dịch |

**`last_session_at` khác `redeemed_at`**: SRS yêu cầu ngày tạo phiên gần nhất, còn `redeemed_at`
là ngày đổi thành công. Một phiên có thể tạo rồi huỷ mà không bao giờ redeem, nên phải tách cột,
không tái dùng.

**`assigned_at` tách khỏi `received_at`** để không đổi ngữ nghĩa cột đang phục vụ app khách hàng,
dù giá trị ban đầu chép từ đó. Cần xác nhận `received_at` đúng là thời điểm gán mã.

### 2.2 Bảng mới

| Bảng | Giữ cái gì | Vì sao hôm nay chưa có | Ghi bởi |
|---|---|---|---|
| `offer_audit` | **Lịch sử tác động**: mỗi dòng là một hành động đã xảy ra với một mã hoặc một giao dịch hoàn tiền — gán mã, tạo phiên, xác nhận, huỷ, hoàn tác, phát sinh cashback | Read model hiện chỉ giữ **trạng thái hiện tại**, không giữ diễn biến. `voucher_ownership.status` cho biết mã đang ở đâu nhưng không cho biết nó đã đi qua những bước nào | `RedemptionAuditConsumer`, `CashbackAuditConsumer`, `VoucherProjectionConsumer` mở rộng |
| `campaign_stock` | **Tồn kho mã của chiến dịch**: tổng số mã, số chưa gán, cờ sinh mã tự động, giá trị mã chung | `voucher_ownership` chỉ có dòng khi mã **đã được gán**. Chiến dịch còn mã chưa phát cho ai thì read model hoàn toàn không biết gì về nó | Consumer tồn kho mã của pp-coupon |

Ba bảng đặt tên không tiền tố, đồng bộ với các bảng read model sẵn có
(`voucher_ownership`, `campaign_offer`, `coupon_display`, `validation_rule`).

`campaign_stock` giải quyết hai yêu cầu mà không bảng nào hiện có trả lời được:

- Phân biệt **"Chưa gán"** (chiến dịch còn mã, khách chưa được nhận) với **"Chưa sinh mã"**
  (chiến dịch chưa có mã nào) — hai giá trị khác nhau ở cột Trạng thái gán mã.
- Dựng **bản ghi giả lập** theo SRS mục 4.4: khách chưa có mã nhưng chiến dịch đang chạy thì vẫn
  phải hiện một dòng với nhãn "Còn mã", "-" hoặc giá trị mã chung.

### 2.3 Chỉ mục đề xuất

| Bảng | Chỉ mục | Vì sao |
|---|---|---|
| `voucher_ownership` | `(customer_source_id, campaign_id)` | Điều kiện lọc bắt buộc của `searchCustomerOffers` |
| `offer_audit` | `(customer_source_id, campaign_id, occurred_at DESC)` | Truy vấn chính của `getAuditTrail`, đã kèm thứ tự sắp xếp |
| `offer_audit` | `(voucher_code)` | Lọc theo mã ưu đãi |

### 2.4 Đường ghi dữ liệu sau khi mở rộng

```mermaid
flowchart LR
    subgraph EXIST["Đã có"]
        C1["customer-event"] --> P1["VoucherProjectionConsumer"]
        C2["promotion_voucher_event"] --> P2["VoucherStatusProjectionConsumer"]
        C3["promotion_campaign_event"] --> P3["CampaignProjectionConsumer"]
        C4["promotion_discount_event"] --> P4["OfferDiscountProjectionConsumer"]
        C5["promotion_coupon_config_event"] --> P5["CouponDisplayProjectionConsumer"]
        C6["promotion_validation_event"] --> P6["ValidationRuleProjectionConsumer"]
    end

    subgraph NEW["Bổ sung"]
        C7["event pp-redemption"] --> P7["RedemptionAuditConsumer"]
        C8["event pp-cashback"] --> P8["CashbackAuditConsumer"]
        R1["REST pp-campaign"] -.-> P3
        R2["REST pp-coupon"] -.-> P5
    end

    P1 --> T1[("voucher_ownership")]
    P2 --> T1
    P3 --> T2[("campaign_offer")]
    P4 --> T2
    P5 --> T3[("coupon_display")]
    P6 --> T4[("validation_rule")]

    P1 --> T5[("offer_audit")]
    P7 --> T5
    P8 --> T5
    P7 --> T1
    P8 --> T1
    P5 --> T6[("campaign_stock")]

    T1 & T2 & T3 & T4 --> V["customer_voucher_view"]

    style NEW fill:#eef,stroke:#66a
    style T5 fill:#efe,stroke:#393
    style T6 fill:#efe,stroke:#393
```

Mũi tên nét đứt là lookup REST lúc xử lý event, theo mẫu `CampaignCreatedAtBackfillService` đang có.

---

## 3. Tổng hợp thay đổi

| Hạng mục | Số lượng |
|---|---|
| Bảng giữ nguyên cấu trúc | 1 (`validation_rule`) |
| Bảng thêm cột | 3 (`voucher_ownership` +6, `campaign_offer` +1, `coupon_display` +1) |
| Bảng mới | 2 |
| View định nghĩa lại | 1 (thêm 8 cột, giữ nguyên 53 cột cũ) |
| Consumer mới | 2 |
| Consumer mở rộng | 3 (`VoucherProjectionConsumer`, `CampaignProjectionConsumer`, `CouponDisplayProjectionConsumer`) |
| Endpoint backfill mới | 2 |
| Changeset Liquibase | tiếp sau `021`, theo mẫu `006-replace-view-customer-voucher.xml` |

**Không xoá hay đổi cột nào đang có** — luồng app khách hàng (API #12, #13, #01) không bị ảnh hưởng
về mặt schema. Vẫn cần chạy hồi quy vì view bị `DROP` rồi `CREATE` lại.

---

## 4. Điểm cần chốt

| # | Vấn đề | Ảnh hưởng |
|---|---|---|
| 1 | Nguồn của `unassigned_code_count`, `auto_generate`, `shared_code` — pp-coupon bổ sung event hay lookup REST? | `campaign_stock` chưa điền được |
| 2 | `received_at` có đúng là thời điểm gán mã không? | `assigned_at` có thể sai ngữ nghĩa |
| 3 | Điều kiện "khách thoả cấu hình segment" của bản ghi giả lập — read model không lưu segment | Có thể phát sinh bảng thứ tư |
| 4 | Ba trạng thái `DISABLING`, `DELETING`, `DELETED` của pp-campaign gom về đâu | Ánh xạ `campaign_status` |

---

## Phụ lục — Lệnh dựng lại tài liệu này

```bash
cd /mnt/data/Data/code/staging_promotion/p2_promotion-vtm-bff

# Bon bang nen va khoa chinh
for f in src/main/java/vn/viettel/vds/promotion/bff/vtm/adapter/out/persistence/entity/*.java; do
  echo "--- $f"; grep -E "@Table|^\s+private " "$f"
done

# Dinh nghia view moi nhat
sed -n '/CREATE VIEW customer_voucher_view/,/;$/p' \
  src/main/resources/db/changelog/changes/020-validation-order-bounds.xml

# Cac changeset da tung dinh nghia lai view
grep -ln "CREATE VIEW customer_voucher_view" src/main/resources/db/changelog/changes/*.xml

# Consumer va topic
grep -rn "topics = " src/main/java/vn/viettel/vds/promotion/bff/vtm/adapter/in/messaging/
```
