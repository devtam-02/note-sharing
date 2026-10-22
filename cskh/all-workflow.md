# Luồng các API trong `CustomerCareController`

Repo `promotion-vtm-bff`, nhánh `tamntt25/cskh-customer-care-api-rest`, commit `a47b8a2` (22/10/2026).
Tiền tố đường dẫn: `{context-path}/api/cskh/v1`. Người gọi: Bankplus BE. Xác thực nhân viên làm ngoài BFF.

| # | API | Đường dẫn | SRS | Bọc envelope |
|---|---|---|---|---|
| 1 | Đặc tả bộ lọc (dropdown) | `GET /api-specs` | 001 phụ lục 5.1 | Không |
| 2 | Tra cứu ưu đãi | `GET /customer-offers` | 003 | Có |
| 3 | Lịch sử tác động | `GET /customer-offers/audit` | 004 | Có |
| 4 | Chi tiết chiến dịch | `GET /campaign-details/{campaignCode}` | 005 | Có |
| 5 | Danh mục sản phẩm | `GET /products` | 001 mục 2.3 (dropdown sản phẩm) | Có |

Quy ước chung:

- Mọi giá trị dạng danh mục viết chữ hoa (`RUNNING`, `PUBLISHED`, `FAIL`…).
- Phân trang dùng `PageData`: trang bắt đầu từ 0, `size` từ 1 đến 100.
- Các lời gọi ra ngoài trong cùng một đợt chạy song song trên bể luồng `campaignDetailExecutor`. Tác vụ luôn được nộp từ luồng phục vụ request, không nộp lồng từ bên trong bể.
- Lỗi tham số trả 400 `INVALID_PARAMS`. Không thấy chiến dịch trả 404. Dịch vụ nguồn chính hỏng trả lỗi dịch vụ phụ thuộc (`DownstreamUnavailableException`).

---

## 1. `GET /api-specs` — đặc tả bộ lọc

Trả tài liệu OpenAPI rút gọn, chỉ còn thao tác `/customer-offers` và các schema nó dùng. Bankplus đọc `enum` và `x-enum-labels` của từng tham số lọc để dựng dropdown. Không gọi dịch vụ nào khác.

```mermaid
sequenceDiagram
    participant BP as Bankplus BE
    participant C as CustomerCareController
    participant SD as springdoc (trong tiến trình)
    BP->>C: GET /api-specs
    C->>SD: openapiJson()
    alt springdoc tắt hoặc lỗi
        C-->>BP: 503
    else
        SD-->>C: đặc tả toàn dịch vụ
        C->>C: giữ path kết thúc bằng /api/cskh/v1/customer-offers<br/>và các schema được $ref tới (duyệt đệ quy)
        C-->>BP: 200 tài liệu OpenAPI (không bọc data)
    end
```

---

## 2. `GET /customer-offers` — tra cứu ưu đãi

**Tham số:**
- `msisdn` (bắt buộc), `voucherCode`, `campaignName`;
- `publishStatus` (`ALL` / `PUBLISHED` / `UNPUBLISHED` / `NOT_GENERATED`);
- `campaignStatus` (**mặc định `RUNNING`**);
- `useStatus`, `offerType`, `productCode`, `page`, `size`.

```mermaid
sequenceDiagram
    autonumber
    participant BP as Bankplus BE
    participant S as CustomerOfferLookupService
    participant CU as pp-customer
    participant SG as pp-segment
    participant CO as pp-coupon
    participant CB as pp-cashback
    participant CA as pp-campaign
    participant VA as pp-validation
    participant PR as pp-product

    BP->>S: msisdn + bộ lọc
    S->>CU: POST /api/v1/customers/validate (mọi dạng số 0xxx/84xxx)
    alt không thấy khách
        S-->>BP: 200 trang rỗng
    end
    par Đợt 1
        S->>SG: GET /api/v1/customers/{customerId}/segments
        S->>CO: GET /api/v1/vouchers?ownerCustomerId (bỏ mã REVOKED)
        S->>CB: giao dịch hoàn tiền (chỉ khi bật cờ)
        S->>CA: GET /api/v2/campaigns?campaignType&status (cache 60s)
    end
    Note over S: Chiến dịch ứng viên = danh sách theo campaignStatus và offerType,<br/>lọc tên chiến dịch (bỏ dấu). pp-campaign lỗi thì trả 502.
    par Đợt 2
        S->>CA: chi tiết chiến dịch của mã đã gán (cache 60s)
        S->>CO: lượt đổi mã theo chiến dịch của mã đã gán
        S->>VA: GET /api/v1/rule-bindings/batch × 3 loại đối tượng (cache 60s)
        S->>VA: GET /v1/rules/{ruleId} mỗi rule (cache 5 phút)
    end
    Note over S: Xét đối tượng theo phân khúc (logic ba trị) và cờ "chỉ người giữ mã"
    opt publishStatus khác PUBLISHED
        par Đợt 3 — chiến dịch thuộc đối tượng mà khách chưa có mã
            S->>CA: chi tiết chiến dịch
            S->>CO: đếm mã chưa gán (cache 60s)
        end
    end
    S->>PR: phân giải tên sản phẩm theo lô
    S->>S: dựng dòng, lọc, sắp xếp, phân trang trong bộ nhớ
    S-->>BP: 200 PageData<CustomerOfferItem>
```

### 2.1 Xét đối tượng theo phân khúc

Đầu vào là binding còn hiệu lực của chiến dịch (loại `DISCOUNT_COUPON`, `CAMPAIGN`, `CASHBACK`), cây rule của từng binding, và segment của khách (gộp định danh với mã).

```mermaid
flowchart TD
    A[Chiến dịch ứng viên] --> B{Đọc được binding?}
    B -- Không --> X[Không thuộc đối tượng]
    B -- Có --> C{Có rule?}
    C -- Không --> OK[Thuộc đối tượng]
    C -- Có --> D{Đọc được mọi rule?}
    D -- Không --> X
    D -- Có --> E{Rule có node customer.segment.*?}
    E -- Không --> OK
    E -- Có --> F{Biết segment của khách?}
    F -- Không --> X
    F -- Có --> G[Chấm logic ba trị:<br/>node segment = đúng/sai, node khác = chưa biết<br/>ALL / ANY / NONE / XOR]
    G --> H{Có rule cho kết quả SAI?}
    H -- Có --> X
    H -- Không --> OK
```

Cờ "chỉ người giữ mã" là có khi cây rule chứa node `redemption.code_holder.*` hoặc `customer.is_owner`.

### 2.2 Dựng dòng

| Nguồn | Điều kiện | `voucherCode` | `publishStatus` | `useStatus` |
|---|---|---|---|---|
| Mã đã gán | Chiến dịch là ứng viên và khách thuộc đối tượng | Mã của khách | `PUBLISHED`; mã đang `UNPUBLISHED` bên nguồn thì `UNPUBLISHED` | Từ giữ chỗ, lượt đổi gần nhất, bộ đếm |
| Hoàn tiền (khi bật cờ) | Như trên | null | null | `PARTICIPATED` |
| Giả lập: mã chung | Thuộc đối tượng, khách chưa có mã, không "chỉ người giữ mã" | Mã chung | `UNPUBLISHED` | `UNUSED` |
| Giả lập: còn mã | Như trên, còn mã chưa gán > 0 | null (hiển thị "Còn mã") | `UNPUBLISHED` | `UNUSED` |
| Giả lập: tự sinh | Như trên, hết mã nhưng chiến dịch tự sinh mã | null (hiển thị "-") | `NOT_GENERATED` | `UNUSED` |
| Không có dòng | Không tra được chi tiết chiến dịch, không đếm được tồn kho, hoặc hết mã mà không tự sinh | | | |

Bản ghi giả lập chỉ được dựng khi `publishStatus` là `ALL`, `UNPUBLISHED` hoặc `NOT_GENERATED`. Khi lọc `PUBLISHED`, bước xét đối tượng chỉ chạy cho chiến dịch của mã đã gán, và không đếm tồn kho.

### 2.3 Lọc, sắp xếp, phân trang

- **Lọc trong bộ nhớ:**
  - `voucherCode`: chứa chuỗi, không phân biệt hoa thường.
  - `campaignName`: chứa chuỗi, bỏ dấu.
  - `publishStatus`, `campaignStatus`, `useStatus`, `offerType`: so khớp chính xác.
  - `productCode`: khớp mã SKU hoặc mã sản phẩm trong phạm vi áp dụng, có xét phần bị loại trừ.
- **Sắp xếp (SRS 2.3):** ngày bắt đầu giảm dần → ngày gán giảm dần → ngày dùng giảm dần → mã chiến dịch tăng dần → mã ưu đãi.

---

## 3. `GET /customer-offers/audit` — lịch sử tác động

**Tham số:**
- `campaignCode`, `msisdn` (cả hai bắt buộc);
- `voucherCode`, `referenceId`;
- `fromTime` / `toTime` dạng `DD/MM/YYYY`, tính trọn ngày theo giờ Việt Nam;
- `page`, `size`.

```mermaid
sequenceDiagram
    autonumber
    participant BP as Bankplus BE
    participant C as Controller
    participant S as OfferAuditTrailService
    participant CA as pp-campaign
    participant CU as pp-customer
    participant CO as pp-coupon
    participant RD as pp-redemption
    participant CB as pp-cashback

    BP->>C: campaignCode, msisdn, bộ lọc
    C->>C: đổi DD/MM/YYYY thành đầu ngày / cuối ngày (Asia/Ho_Chi_Minh)
    alt ngày không có thật hoặc toTime nhỏ hơn fromTime
        C-->>BP: 400
    end
    C->>S: AuditQuery
    S->>CA: GET /api/v2/campaigns?code&campaignType (DISCOUNT_COUPON rồi CASHBACK, lọc đúng mã)
    alt không thấy chiến dịch
        S-->>BP: 404
    end
    S->>CU: tra khách theo số
    alt không thấy khách
        S-->>BP: 200 trang rỗng
    end
    par
        S->>CO: mã của khách trong chiến dịch
        S->>CO: lượt đổi mã của khách trong chiến dịch
        S->>RD: GET /v1/redemption/sessions (mọi dạng số thuê bao)
        S->>CB: giao dịch hoàn tiền (khi bật cờ), lọc đúng chiến dịch
    end
    S->>S: gộp bốn nguồn (nguồn trước thắng khi trùng hành động + mã phiên + mã ưu đãi)
    S->>S: lọc voucherCode, referenceId, khoảng thời gian, sắp xếp mới nhất trước, phân trang
    S-->>BP: 200 PageData<AuditEntry>
```

**Hành động sinh ra từ mỗi nguồn:**

| Nguồn | Hành động |
|---|---|
| Phiên đổi thưởng (pp-redemption) | `SESSION_CREATED`, `SESSION_CONFIRMED` (`result` là `SUCCESS` hoặc `FAIL`), `SESSION_CANCELLED`, `REDEMPTION_ROLLED_BACK` |
| Lượt đổi mã (pp-coupon) | `SESSION_CONFIRMED`, `REDEMPTION_ROLLED_BACK` |
| Mã đã gán (pp-coupon) | `VOUCHER_ASSIGNED` |
| Giao dịch hoàn tiền (pp-cashback) | `TRANSACTION_QUALIFIED`, `TRANSACTION_CONFIRMED`, `CASHBACK_CREATED` |

Nguồn phụ lỗi chỉ làm thiếu nhóm dòng của nguồn đó. API này không lọc theo phân khúc.

---

## 4. `GET /campaign-details/{campaignCode}` — chi tiết chiến dịch

```mermaid
sequenceDiagram
    autonumber
    participant BP as Bankplus BE
    participant S as CampaignDetailAggregationService
    participant CA as pp-campaign
    participant CO as pp-coupon
    participant PE as pp-pricing-engine
    participant PR as pp-product

    BP->>S: campaignCode
    S->>CA: GET /api/v2/campaigns?code (đổi mã thành định danh)
    alt không thấy
        S-->>BP: 404
    end
    S->>CA: GET /api/v2/campaigns/{id}
    alt pp-campaign hỏng
        S-->>BP: 502
    end
    par Nguồn phụ, lỗi thì để trống trường tương ứng
        S->>CO: cấu hình mã theo chiến dịch
        S->>PE: công thức giảm giá theo chiến dịch
        S->>PR: tên sản phẩm của phạm vi áp dụng
    end
    S->>S: CampaignDetailMapper
    S-->>BP: 200 CampaignDetailResponse (16 trường SRS 005)
```

**Nguồn của từng trường:**

| Trường | Nguồn |
|---|---|
| `campaignCode`, `campaignName`, `campaignStatus`, `periodFrom`, `periodTo` | pp-campaign |
| `periodTimeLimit` | Tổng hợp từ khung giờ và ngày trong tuần của pp-campaign (phụ lục 5.2) |
| `campaignType` | Loại mã của pp-coupon (dự phòng: pp-campaign). `BULK_CODE` thành `BATCH_CODE`, `SINGLE_CODE` thành `COMMON_CODE`. Hoàn tiền trả null |
| `discountType` | Nhóm giảm giá của pp-pricing-engine (dự phòng: bản chụp ở pp-campaign). `ORDER` thành `TRANSACTION_DISCOUNT`, `PRODUCT` thành `PRODUCT_DISCOUNT`, `FREE_SHIPPING` thành `NO_TRANSACTION_FEE` |
| `discountMethod` | `AMOUNT` giữ nguyên, `PERCENT` thành `PERCENTAGE`. Null khi `NO_TRANSACTION_FEE` |
| `percentageOff`, `amountOff` | pp-pricing-engine (dự phòng: pp-campaign) |
| `applicableProducts` | Phạm vi áp dụng ở pp-campaign, tên lấy từ pp-product, bỏ phần bị loại trừ |
| `voucherName`, `brandName`, `detailInfo` | pp-coupon (dự phòng: bản chụp hiển thị ở pp-campaign) |
| `usageGuideUrl` | Như trên, đổi khoá ảnh thành đường dẫn xem ảnh |

---

## 5. `GET /products` — danh mục sản phẩm

Chuyển tiếp sang `GET /api/v1/products` của pp-product, dùng làm nguồn cho dropdown "Sản phẩm áp dụng".

```mermaid
sequenceDiagram
    participant BP as Bankplus BE
    participant S as CustomerCareSearchService
    participant PR as pp-product
    BP->>S: productName, skuName, onlyProduct (mặc định true), sort, page, size
    S->>S: chuỗi rỗng thành không lọc, cắt khoảng trắng
    S->>PR: GET /api/v1/products
    alt pp-product lỗi hoặc thân rỗng
        S-->>BP: 502
    else
        PR-->>S: trang sản phẩm
        S-->>BP: 200 PageData<CustomerCareCatalogProduct> (nguyên trạng)
    end
```

- `sort` chỉ nhận cột `name`, `createdAt`, `updatedAt` (dạng `cột,hướng`).
- Để lọc ưu đãi theo chiến dịch giới hạn sản phẩm, lấy `productCode` của kết quả. Với chiến dịch giới hạn theo SKU thì gọi `onlyProduct=false` và lấy `skuCode`.

---

## 6. Cache trong tiến trình

| Dữ liệu | Thời gian sống | Ghi chú |
|---|---|---|
| Danh sách chiến dịch theo loại và trạng thái | 60 giây | Tối đa 20 trang × 100 chiến dịch cho mỗi loại |
| Chi tiết chiến dịch | 60 giây | Không cache kết quả rỗng |
| Binding rule theo chiến dịch | 60 giây | Không cache lỗi |
| Cây rule | 5 phút | Không cache lỗi |
| Số mã chưa gán | 60 giây | Không cache lỗi |
