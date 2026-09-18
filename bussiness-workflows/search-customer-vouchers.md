# API #13 — Search Customer Vouchers: luồng xử lý đầy đủ

> `GET /promotion/promotion-vtm-bff/api/v1/vtm/customer-vouchers`
> Entry point: `VtmVoucherController.search` (`adapter/in/web/VtmVoucherController.java:57`)
> Tài liệu dựng theo **cấu hình trong `src/main/resources/application.properties`** (profile mặc định, không phải `application-uat.properties`).
> Tài liệu chị em: [`api-12-get-customer-voucher-detail-flow.md`](api-12-get-customer-voucher-detail-flow.md)

---

## 1. Tóm tắt một dòng

Request đi qua `VtmAuthFilter` verify JWT RS256 lấy msisdn từ claim `usr`, tới `VtmVoucherService` chuẩn hoá tham số, qua port `VoucherApiClient` tới **`CustomerVoucherViewAdapter`** đọc thẳng VIEW `customer_voucher_view` trên MariaDB bằng JPA Specification động dựng từ danh mục tab, map sang `OfferItem`, trả `SearchVouchersResponse` bọc trong `ResponseTemplate`.

**Không có lời gọi downstream đồng bộ nào trong luồng chính**, trừ một nhánh phụ (`pp-product`, có cache) chỉ kích hoạt cho voucher áp dụng cho mọi dịch vụ.

---

## 2. Cấu hình quyết định luồng

| Khoá | Giá trị | Ảnh hưởng |
|---|---|---|
| `spring.application.context-path` | `/promotion/promotion-vtm-bff` | Tiền tố URL của controller |
| `bff.voucher.read-model.enabled` | `true` | **Chọn `CustomerVoucherViewAdapter`** (đọc DB) làm implementation `VoucherApiClient` |
| `bff.mock.enabled` | `false` | `VoucherApiAdapter` (Feign) vẫn được tạo bean nhưng bị `@Primary` của read-model che; `MockVoucherApiClient` không được tạo |
| `bff.vtm.auth.enabled` | `true` | Bật `VtmAuthFilter` |
| `bff.vtm.auth.public-key` | RSA X.509 base64 | Verify chữ ký RS256 |
| `bff.eligible.expire-warning-days` | **không khai, mặc định `7`** | Ngưỡng "sắp hết hạn", đồng thời là `expireWarningDate` echo về client |
| `bff.voucher.search.tabs[*]` | **không khai, dùng `VoucherTabCatalog.builtInDefaults(7)`** | Thanh tab là `all` cộng `expiring_soon` dựng sẵn trong code |
| `bff.voucher.campaign-usable-statuses` | **không khai, mặc định `RUNNING`** | Chỉ voucher thuộc campaign `RUNNING` được coi là dùng được |
| `bff.image.public-base-url` | `https://api24cdn.vtmoney.vn/promotion-campaign` | Base URL dựng link logo và banner |
| `spring.datasource.url` | MariaDB, 3 node, schema `promotion_vtm_bff` | Nguồn đọc read model, xem §5 |
| `spring.datasource.hikari.maximum-pool-size` | `20` | Trần đồng thời thực tế của API này |
| `spring.jpa.open-in-view` | `false` | Session đóng khi ra khỏi `@Transactional(readOnly = true)` |
| `app.downstream.product.base-url` | `http://10.207.201.204:8083/uatmm` | `pp-product` cho nhánh `applies_to_all` |
| `promix.cache.redisson.*` | CLUSTER, TTL 30m | Cache danh mục dịch vụ `bff-vtm-service-catalog` |

Ba khoá "không khai" ở trên là điểm dễ hiểu nhầm nhất: hành vi tab và ngưỡng hết hạn **không** đến từ file properties mà từ giá trị mặc định hard-code.

---

## 3. Sơ đồ luồng chính

```mermaid
sequenceDiagram
    autonumber
    participant App as App VTM
    participant F as VtmAuthFilter
    participant V as VtmTokenVerifier
    participant C as VtmVoucherController
    participant S as VtmVoucherService
    participant A as CustomerVoucherViewAdapter
    participant T as VoucherTabCatalog
    participant SP as VoucherSearchSpecifications
    participant R as CustomerVoucherViewRepository
    participant DB as MariaDB customer_voucher_view
    participant M as CustomerVoucherViewMapper
    participant SC as ServiceCatalogAdapter

    App->>F: GET customer-vouchers voi keyword serviceCode tab page size
    F->>F: shouldNotFilter kiem tra path chua api v1
    F->>V: verify token RS256 va han dung
    alt token thieu hoac sai chu ky hoac het han
        V-->>F: VtmTokenException
        F-->>App: 401 UNAUTHORIZED
    end
    V-->>F: VtmUserInfo gom sub va usr la msisdn
    F->>F: VtmUserContextHolder set userInfo
    F->>C: doFilter

    C->>C: Bean Validation Size Min Max va VoucherTab
    note right of C: VoucherTab goi VoucherTabCatalog isAllowedTab<br/>sai thi tra ve 400 INVALID_PARAMS
    C->>S: useCase search

    S->>S: resolveCustomerId lay claim usr
    note right of S: rong thi nem CustomerIdRequiredException 422
    S->>S: keyword va serviceCode rong thi doi thanh null
    note right of S: keyword KHONG ep tab ve all
    S->>A: voucherApi search qua port VoucherApiClient

    A->>A: lay now va locale tu Accept-Language
    A->>T: selectedTab tra ve tab hoac all
    A->>T: conditionsOf selectedTab
    T-->>A: danh sach TabCondition

    A->>SP: base gom customerId groupType keyword serviceCode
    A->>SP: tab gom dieu kien cua tab dang chon
    A->>SP: displayOrder gom khoa sap xep
    A->>R: findAll voi spec va PageRequest
    R->>DB: Q1 SELECT tren customer_voucher_view co WHERE ORDER BY LIMIT OFFSET
    DB-->>A: Page cua CustomerVoucherViewEntity
    opt khong bo qua duoc count, xem muc 6.5
        R->>DB: Q2 SELECT COUNT tren customer_voucher_view cho totalElements
    end

    loop moi tab dang bat, mac dinh 2 tab
        A->>R: count voi base va dieu kien cua tab do
        R->>DB: Q3 va Q4 SELECT COUNT tren customer_voucher_view
    end
    note over R,DB: Toan bo luong chinh chi cham DUY NHAT<br/>doi tuong promotion_vtm_bff.customer_voucher_view
    A->>T: buildTabs tra ve danh sach TabInfo

    loop moi row cua trang
        A->>M: toItem voi serviceApplies true
        M->>M: releaseExpiredReservation roi CampaignStatusPolicy roi effectiveStatus
        opt voucher ap dung cho moi dich vu
            M->>SC: loadAllServices co cache Redis
            SC-->>M: danh muc dich vu, rong neu pp-product loi
        end
        M-->>A: OfferItem
    end
    A->>A: sort lai bang VoucherDisplayOrder de chen khoa priority
    A-->>S: SearchVouchersResponse
    S-->>C: SearchVouchersResponse
    C-->>App: 200 ResponseTemplate boc SearchVouchersResponse
    F->>F: finally VtmUserContextHolder clear
```

---

## 4. Chọn implementation của port `VoucherApiClient`

```mermaid
flowchart TD
    Q["VtmVoucherService goi voucherApi.search"] --> D{"bff.voucher.read-model.enabled"}
    D -->|"true la cau hinh hien tai"| RM["CustomerVoucherViewAdapter<br/>Primary va ConditionalOnProperty<br/>doc MariaDB customer_voucher_view"]
    D -->|"false"| D2{"bff.mock.enabled"}
    D2 -->|"false la cau hinh hien tai"| FE["VoucherApiAdapter<br/>Feign toi pp-customer voucher_warehouse<br/>loc keyword va tab trong bo nho"]
    D2 -->|"true"| MK["MockVoucherApiClient"]

    style RM fill:#1f6f3f,color:#ffffff
```

Với `application.properties` hiện tại, **nhánh đang chạy là `CustomerVoucherViewAdapter`**. Hai nhánh còn lại được mô tả ở §11.

---

## 5. Kho dữ liệu: loại DB, tên database, bảng

### 5.1 Hệ quản trị và kết nối

| Hạng mục | Giá trị |
|---|---|
| Loại DB | **MariaDB** (quan hệ) |
| Driver | `org.mariadb.jdbc.Driver` |
| Dialect Hibernate | `org.hibernate.dialect.MariaDBDialect` |
| JDBC URL | `jdbc:mariadb:sequential://10.208.81.119:4006,10.208.81.120:4006,10.208.81.121:4006/promotion_vtm_bff` |
| **Tên database (schema)** | **`promotion_vtm_bff`** |
| Cụm | 3 node, cổng `4006`. Tiền tố `sequential://` nghĩa là driver thử lần lượt theo đúng thứ tự khai báo cho tới khi nối được, không cân bằng tải |
| Connection pool | HikariCP, tên `promotion-vtm-bff`, min-idle `5`, **max `20`**, connection-timeout `30s`, max-lifetime `300s` |
| Tầng truy cập | Spring Data JPA và Hibernate, truy vấn dựng bằng Criteria API qua `JpaSpecificationExecutor` |
| Quản lý schema | `spring.jpa.hibernate.ddl-auto=none` và `spring.liquibase.enabled=false`, tức **ứng dụng không tạo hay sửa schema**, changelog trong repo chỉ là mô tả |

### 5.2 API #13 chạm vào đối tượng nào

Đúng **một** đối tượng, và nó **không phải bảng** mà là **VIEW**:

```
promotion_vtm_bff.customer_voucher_view
```

Ánh xạ tại `CustomerVoucherViewEntity` bằng `@Table(name = "customer_voucher_view")` kèm `@Immutable`.

**Ma trận CRUD của API #13:**

| Thao tác | Có thực hiện | Đối tượng |
|---|---|---|
| CREATE | không | — |
| READ | **có** | `promotion_vtm_bff.customer_voucher_view` |
| UPDATE | không | — |
| DELETE | không | — |

API #13 là **đọc thuần**. Ba lớp chặn ghi cộng dồn:

1. `@Transactional(readOnly = true)` trên `CustomerVoucherViewAdapter.search`
2. `@Immutable` của Hibernate trên entity, Hibernate bỏ qua mọi dirty-check
3. Bản thân `customer_voucher_view` là view join bốn bảng nên MariaDB không cho phép ghi

**Số câu lệnh SQL mỗi request** với bộ tab mặc định (2 tab): **3 hoặc 4**, tất cả đều trên `customer_voucher_view`. Danh sách đầy đủ kèm mệnh đề thật ở §6.5.

Thêm một tab cấu hình là thêm một câu COUNT trên toàn tập của khách, cho mỗi request.

### 5.3 Bảng cơ sở phía sau view và ai ghi vào chúng

View `customer_voucher_view` không lưu dữ liệu. Nó join bốn bảng, và bốn bảng đó được ghi bởi **luồng projection Kafka**, hoàn toàn tách khỏi đường request của API #13.

```mermaid
flowchart TD
    subgraph W["Duong GHI, projection Kafka"]
        K1["pp-customer va pp-coupon"] --> A1["VoucherOwnershipStoreAdapter<br/>VoucherExpiryBackfillStoreAdapter"]
        K2["pp-campaign va pp-pricing-engine"] --> A2["CampaignOfferStoreAdapter<br/>CampaignOfferBackfillAdapter"]
        K3["pp-coupon"] --> A3["CouponDisplayStoreAdapter<br/>CouponDisplayBackfillAdapter"]
        K4["pp-validation"] --> A4["ValidationRuleStoreAdapter"]
        A1 --> T1[("voucher_ownership")]
        A2 --> T2[("campaign_offer")]
        A3 --> T3[("coupon_display")]
        A4 --> T4[("validation_rule")]
    end

    T1 --> VIEW[("customer_voucher_view")]
    T2 --> VIEW
    T3 --> VIEW
    T4 --> VIEW

    subgraph R["Duong DOC, API 12 va 13"]
        VIEW --> API["CustomerVoucherViewRepository<br/>chi SELECT"]
    end

    style VIEW fill:#1f4f6f,color:#ffffff
```

| Bảng | Adapter ghi | Thao tác thực tế | Nguồn dữ liệu |
|---|---|---|---|
| `voucher_ownership` | `VoucherOwnershipStoreAdapter`, `VoucherExpiryBackfillStoreAdapter` | `INSERT` và `UPDATE` qua `repository.save`, thêm hai câu `@Modifying` bulk `UPDATE` | pp-customer (`CustomerVoucher*`), pp-coupon (`VoucherEvent`) |
| `campaign_offer` | `CampaignOfferStoreAdapter`, `CampaignOfferBackfillAdapter` | `INSERT` và `UPDATE` | pp-campaign (`CampaignEvent`), pp-pricing-engine (`DiscountEvent`) |
| `coupon_display` | `CouponDisplayStoreAdapter`, `CouponDisplayBackfillAdapter` | `INSERT`, `UPDATE`, và **`DELETE`** qua `deleteById(couponConfigId)` | pp-coupon (`CouponConfigEvent`) |
| `validation_rule` | `ValidationRuleStoreAdapter` | `INSERT` và `UPDATE` | pp-validation (`ValidationEvent`) |

`coupon_display` là bảng **duy nhất** trong read model có thao tác DELETE.

### 5.4 Kho dữ liệu thứ hai trong luồng

Ngoài MariaDB, đường request của #13 còn chạm tới **Redis/Valkey** qua Redisson, nhưng chỉ ở một nhánh phụ:

| Hạng mục | Giá trị |
|---|---|
| Chế độ | `promix.cache.redisson.mode=CLUSTER` |
| Node | `10.208.81.136`, `.137`, `.138`, mỗi node hai cổng `7279` và `7379` |
| Tên cache | `bff-vtm-service-catalog` |
| Khoá | một khoá duy nhất `'all-services'` |
| TTL | `promix.cache.redisson.default-ttl=30m` |
| Khi nào chạm | chỉ khi `CustomerVoucherViewMapper` bung danh mục cho voucher áp dụng cho mọi dịch vụ, cache miss thì gọi `pp-product` |
| Codec | `org.redisson.codec.JsonJacksonCodec` |

Danh sách rỗng **không** được đưa vào cache (`unless` trên `@Cacheable`), tránh ghim "không có dịch vụ nào" suốt vòng đời cache khi pp-product chập chờn.

Ngoài hai kho trên, luồng #13 không chạm datastore nào khác. Kafka chỉ nuôi read model chứ không nằm trong đường request.

---

## 6. Chi tiết từng chặng

### 6.1 `VtmAuthFilter` — xác thực (`adapter/in/web/auth/`)

| Thuộc tính | Nguồn | Giá trị hiệu lực |
|---|---|---|
| Đăng ký | `VtmAuthConfig.vtmAuthFilterRegistration` | urlPatterns `/*`, order `HIGHEST_PRECEDENCE + 10` |
| Path được bảo vệ | `bff.vtm.auth.protected-path-prefixes` (không khai) | `["/api/v1/"]`, so khớp bằng `path.contains(prefix)` |
| Header | `bff.vtm.auth.token-header` | `Authorization` |
| Tiền tố | `bff.vtm.auth.token-prefix` | `Bearer ` |
| Verify issuer | `bff.vtm.auth.verify-issuer` (không khai) | `false` |
| Base64 lenient | `bff.vtm.auth.lenient-base64-signature` (không khai) | `false` |

Thành công thì đặt `VtmUserInfo` vào `VtmUserContextHolder` (ThreadLocal), luôn `clear()` ở `finally`.
Thất bại trả **401** body `ResponseTemplate` với `code` là `UNAUTHORIZED` và message `"Yêu cầu chưa được xác thực"`.

### 6.2 Controller — validate tham số

| Tham số | Ràng buộc | Mặc định |
|---|---|---|
| `keyword` | `@Size(max = 255)` | — (null) |
| `serviceCode` | `@Size(max = 64)` | — (null) |
| `tab` | `@Size(max = 32)` kèm `@VoucherTab` | — (dùng tab mặc định `all`) |
| `page` | `@Min(0)` | `0` |
| `size` | `@Min(1) @Max(100)` | `10` |

`@VoucherTab` gọi `VoucherTabValidator` rồi `VoucherTabCatalog.isAllowedTab()`: hợp lệ khi rỗng hoặc thuộc danh mục tab **đang cấu hình**, không hard-code. Vi phạm bất kỳ sẽ rơi vào `InvalidParamsExceptionHandler` và trả **400** với `code` là `INVALID_PARAMS` kèm `errors[]` nêu tên trường.

### 6.3 `VtmVoucherService` — orchestration mỏng

```
cid                   = VtmUserContextHolder.get().msisdn()   // claim "usr", KHÔNG phải "sub"
normalizedKeyword     = blank ? null : trim(keyword)          // không có độ dài tối thiểu
normalizedServiceCode = blank ? null : trim(serviceCode)      // "" sẽ tạo LIKE '%,,%' nên phải về null
```

Hai điểm hợp đồng cần nhớ:
- **Client không được truyền `customerId`**, luôn lấy từ token.
- **`keyword` không đổi tab.** Trước v1.19 service ép `tab=all` khi có keyword, quy tắc đó đã bỏ.

`msisdn` rỗng sẽ ném `CustomerIdRequiredException` (extends `BusinessRuleException` của promix), `PlatformExceptionHandler` trả **422** với `code` là `PP-BFF-VTM-2006`.

### 6.4 `CustomerVoucherViewAdapter.search` — trái tim của API

`@Transactional(readOnly = true)`. Bốn việc, theo thứ tự:

1. **Dựng specification**
   ```
   base     = VoucherSearchSpecifications.base(cid, "MY", lower(keyword), serviceCode)
   selected = base
            AND tab(tabCatalog.conditionsOf(selectedTab), now)
            AND displayOrder(tabOrdersOf(selectedTab), now, campaignStatusPolicy.usableStatuses())
   ```
2. **Truy vấn trang**: `repository.findAll(selected, PageRequest.of(page, size, Sort.unsorted()))`
   `Sort.unsorted()` là **cố ý**. Spring Data sẽ ghi đè `orderBy` của specification nếu `Pageable` mang sort, mà khoá nhóm ưu đãi là biểu thức `CASE` không khai được bằng tên thuộc tính.
3. **Đếm badge**: mỗi tab đang bật là **một câu `COUNT`** trên `base AND tab(conditions của tab đó)`, bỏ qua phân trang và bỏ qua tab đang chọn.
4. **Map và sắp lại**: `mapper.toItem(...)` rồi `.sorted(VoucherDisplayOrder.DISPLAY_ORDER)` trên nội dung *một trang*, chỉ để chèn khoá phụ `priority` (parse từ JSON metadata, không có cột).

### 6.5 Truy vấn DB phát sinh trong luồng chính

Liệt kê theo đúng thứ tự thực thi. **Mọi câu đều chạy trên `promotion_vtm_bff.customer_voucher_view`** — luồng chính của API #13 không chạm bảng nào khác, không join thêm, không gọi stored procedure.

| # | Chạy khi nào | Câu lệnh (rút gọn) | Sinh bởi |
|---|---|---|---|
| **Q1** | luôn | `SELECT <các cột của view> FROM customer_voucher_view WHERE <base> AND <tab> ORDER BY <displayOrder> LIMIT :size OFFSET :page*:size` | `CustomerVoucherViewRepository.findAll(spec, PageRequest)` |
| **Q2** | có điều kiện, xem dưới | `SELECT COUNT(*) FROM customer_voucher_view WHERE <base> AND <tab>` | count query của `Page`, Spring Data tự phát |
| **Q3** | luôn | `SELECT COUNT(*) FROM customer_voucher_view WHERE <base>` (tab `all` không có điều kiện riêng nên phần `<tab>` rỗng) | `repository.count(...)` trong vòng lặp badge |
| **Q4** | luôn | `SELECT COUNT(*) FROM customer_voucher_view WHERE <base> AND <expiring_soon>` | như trên |

Nội dung thật của `<base>`, `<tab>`, `<displayOrder>` ở §6.6 và §6.7.

**Q2 có thể bị bỏ qua.** `SimpleJpaRepository.findAll(Specification, Pageable)` gói kết quả bằng `PageableExecutionUtils.getPage`, và lớp này chỉ phát count khi thật sự cần:

```java
// rút gọn từ PageableExecutionUtils
if (pageable.getOffset() == 0 && pageable.getPageSize() > content.size())
    total = content.size();                         // trang đầu chưa đầy, KHÔNG count
else if (content.size() != 0 && pageable.getPageSize() > content.size())
    total = pageable.getOffset() + content.size();  // trang cuối chưa đầy, KHÔNG count
else
    total = countSupplier.getAsLong();              // mới phát Q2
```

Ví của khách có ít hơn `size` voucher chỉ tốn **3** round-trip, ví nhiều trang tốn **4**.

**Q3 và Q4 luôn chạy**, kể cả khi client đang xem tab `expiring_soon`: badge phải hiển thị số của *mọi* tab, không chỉ tab đang chọn. Số câu COUNT bằng đúng số tab đang bật, nên bật thêm tab là cộng thẳng vào chi phí mỗi request.

**Cả 3 hoặc 4 câu nằm trong một transaction `readOnly` duy nhất** (`@Transactional(readOnly = true)` trên `CustomerVoucherViewAdapter.search`), dùng một connection Hikari giữ suốt phương thức. Vì `spring.jpa.open-in-view=false`, connection được trả về pool ngay khi `search()` trả, trước khi Jackson serialize response.

### Các chặng KHÔNG chạm DB

Liệt kê tường minh để khỏi phải đoán khi đọc sơ đồ:

| Chặng | Lấy dữ liệu từ đâu |
|---|---|
| `VtmAuthFilter` và `VtmTokenVerifier` | public key RSA tĩnh trong `bff.vtm.auth.public-key`. Không tra DB, không gọi service xác thực nào |
| `VoucherTabCatalog` | danh mục tab dựng một lần trong constructor lúc khởi động, giữ nguyên trong bộ nhớ |
| `CampaignStatusPolicy` | whitelist parse từ properties lúc khởi động |
| `ImagePreviewUrlResolver`, `UsageGuideHtmlRenderer` | thuần xử lý chuỗi |
| `VoucherDisplayOrder` | comparator in-memory trên nội dung trang |
| `ServiceCatalogPort.loadAllServices` | **Redis** (cache `bff-vtm-service-catalog`), cache miss thì gọi HTTP sang `pp-product`. Không phải MariaDB |

### 6.6 Mệnh đề WHERE sinh ra

```sql
-- base (luôn có)
customer_source_id = :cid
AND group_type = 'MY'

-- nếu keyword != null  (đã lower-case ở adapter)
AND ( title_normalized         LIKE CONCAT('%', :kw, '%')
   OR merchant_name_normalized LIKE CONCAT('%', :kw, '%')
   OR LOWER(voucher_code)      =  :kw )          -- mã voucher khớp TUYỆT ĐỐI

-- nếu serviceCode != null
AND ( applies_to_all = 1 OR applicable_service_codes LIKE CONCAT('%,', :svc, ',%') )
AND ( excluded_service_codes IS NULL
   OR excluded_service_codes NOT LIKE CONCAT('%,', :svc, ',%') )
```

Ký tự đại diện người dùng gõ (`%`, `_`) **không** được thoát, giữ nguyên hành vi của câu JPQL cũ.

Phần tab (bộ mặc định, tab `expiring_soon`, `now` là thời điểm request, `expireWarningDays` là 7):

```sql
AND expiration_date <= :now + INTERVAL 7 DAY     -- endDate LTE 7 DAYS
AND expiration_date >= :now                      -- endDate GTE 0 DAYS
AND status IN ('ACTIVE','AVAILABLE','PARTIALLY_USED')
AND ( campaign_status IS NULL
   OR campaign_status NOT IN ('INITIALIZING','ACTIVE','ENABLING','DISABLING','UPDATING',
                              'PAUSED','EXPIRED','ERROR','DELETING','DELETED','DISABLED') )
```

Quy ước NULL, do `VoucherSearchSpecifications.toPredicate` áp:
- **Phép khẳng định** (`EQ`, `IN`, `CONTAINS`, so sánh lớn bé): NULL bị loại theo ngữ nghĩa SQL ba trạng thái. Vì thế tab `expiring_soon` không cần khai `endDate IS NOT NULL`, voucher vô thời hạn tự rớt.
- **Phép loại trừ** (`NEQ`, `NOT_IN`): NULL **được cho qua** (`path IS NULL OR NOT IN (...)`), nếu không thì mọi voucher chưa nhận event trạng thái chiến dịch sẽ biến mất.
- `include-null=true` bọc thêm `OR path IS NULL`, chỉ khai được cho phép khẳng định (`VoucherTabCatalog` chặn từ lúc khởi động).

### 6.7 Mệnh đề ORDER BY sinh ra

`VoucherSearchSpecifications.displayOrder` chỉ áp cho câu SELECT, câu COUNT trả `Long` nên bỏ qua:

```sql
ORDER BY
  /* 1. nhóm ưu đãi, SRS Mức 2, phải là khoá NGOÀI CÙNG */
  CASE
    WHEN campaign_status IS NOT NULL AND TRIM(campaign_status) <> ''
         AND UPPER(campaign_status) NOT IN ('RUNNING')            THEN 3   -- EXPIRED
    WHEN UPPER(status) = 'REDEEMED'                               THEN 2
    WHEN ( UPPER(status) = 'ACTIVE'
        OR (UPPER(status)='RESERVED' AND reserved_until IS NOT NULL AND reserved_until < :now) )
         AND (expiration_date IS NULL OR expiration_date >= :now) THEN 1   -- ACTIVE
    ELSE 3
  END ASC,
  /* 2. voucher vô thời hạn xuống cuối NHÓM, MariaDB xếp NULL lên đầu với ASC */
  CASE WHEN expiration_date IS NULL THEN 1 ELSE 0 END ASC,
  /* 3. khoá sort của tab, bộ mặc định là endDate ASC */
  expiration_date ASC,
  /* 4. tie-breaker giữ phân trang ổn định */
  created_at DESC
```

Danh sách `('RUNNING')` đến thẳng từ `bff.voucher.campaign-usable-statuses`, cùng một nguồn với `CampaignStatusPolicy` mà mapper dùng, để thứ tự SQL không lệch trạng thái khách nhìn thấy.

> **Lưu ý về tính đúng của phân trang:** thứ tự nhóm được dựng ở SQL nên phân trang đúng trên toàn tập (PROM-1398). Nhưng comparator `VoucherDisplayOrder.DISPLAY_ORDER` chạy lại **trong phạm vi một trang** để chèn `priority`. Nếu `priority` phân biệt được hai hàng mà SQL coi là bằng nhau, thứ tự giữa các trang có thể không nhất quán tuyệt đối.

---

## 7. Danh mục tab

Vì `bff.voucher.search.tabs[*]` không được khai, `VoucherTabCatalog` dùng `builtInDefaults(7)`:

| code | default | order | filter | sort |
|---|---|---|---|---|
| `all` | có | 1 | (không có) | `endDate ASC` |
| `expiring_soon` | không | 2 | `endDate LTE 7 DAYS`, `endDate GTE 0 DAYS`, `status IN (ACTIVE,AVAILABLE,PARTIALLY_USED)`, `campaignStatus NOT_IN (11 trạng thái)` | `endDate ASC` |

Nếu về sau có khai cấu hình, `VoucherTabCatalog` validate **ngay trong constructor**:
- `code` bắt buộc, không trùng, `label-i18n` không rỗng
- đúng **một** tab `default-tab=true`, và tab đó phải `enabled=true`
- `field` phải thuộc whitelist `TabField` (14 field), `operator` thuộc `TabOperator` (9 toán tử), và operator phải hợp kiểu với field
- `unit` chỉ nhận `DAYS` và chỉ cho field kiểu `INSTANT`
- `include-null` không dùng với `NEQ` hay `NOT_IN`
- `value` phải parse được ngay lúc khởi động

Cấu hình sai sẽ ném `IllegalStateException`, **nhưng** `VoucherTabCatalog` bắt lại (`catch (Exception)`) và lùi về bộ mặc định kèm `log.warn`, nên service vẫn start. `VoucherSearchPropertiesConfig` cũng bind thủ công có try-catch vì cú pháp placeholder `-${...}` làm Spring Cloud Config trên staging vỡ.

`TabField` là ranh giới an toàn: tên field trong properties không bao giờ đi thẳng vào câu query (Criteria API luôn bind tham số), và chỉ trỏ được vào cột đã whitelist.

---

## 8. Map row sang `OfferItem` (`CustomerVoucherViewMapper.toItem`)

```mermaid
flowchart TD
    R["row cua customer_voucher_view"] --> A["VoucherUsability.releaseExpiredReservation<br/>RESERVED da qua reserved_until thi coi la ACTIVE"]
    A --> B{"CampaignStatusPolicy.isDisabling<br/>campaign_status ngoai whitelist RUNNING"}
    B -->|"co"| E1["ep ve EXPIRED"]
    B -->|"khong"| C
    E1 --> C["VoucherStatusLabelSupport.effectiveStatus<br/>ACTIVE RESERVED SUSPENDED ma qua han thi thanh EXPIRED<br/>gom ve dung 3 gia tri hop dong"]
    C --> D["fixedLabel theo locale<br/>ACTIVE la Su dung, REDEEMED la Da su dung, EXPIRED la Da het han"]
    C --> M["metadata gom usable, disabledReason, displayMode<br/>cong campaign_metadata va voucher_metadata"]
    R --> P["applicable_products dang JSON"]
    P --> Q{"applies_to_all bang 1<br/>va discount_method la APPLY_TO_ORDER<br/>va khong co item INCLUDED"}
    Q -->|"co"| S["ServiceCatalogPort.loadAllServices<br/>goi pp-product qua cache Redis"]
    Q -->|"khong"| T["dung danh sach da luu, bo item EXCLUDED"]
    R --> U["logo, banner, usage_guide qua ImagePreviewUrlResolver<br/>object key ghep voi bff.image.public-base-url"]
```

Chi tiết đáng nhớ:
- `expiredTimeNumber` bằng `ChronoUnit.DAYS.between(now, expiration_date)`, **có thể âm** với voucher đã hết hạn.
- `voucher.id` phơi ra là `voucher_id`. Nhưng #01 và một số luồng phơi `campaign_id`, nên API #12 phải tra cả hai.
- `serviceApplies` luôn `true` ở #13 vì `serviceCode` đã được lọc ở SQL.
- Ngày tháng format theo `Asia/Ho_Chi_Minh` dù `spring.jackson.time-zone=UTC`, hai thứ độc lập vì mapper tự format ra `String`.
- `guideline` trả về **đoạn HTML** đã bọc thẻ (`UsageGuideHtmlRenderer`), không phải link ảnh (PROM-1419).

Locale: `VtmLocaleConfig` đặt `AcceptHeaderLocaleResolver` với default `vi-VN` và chỉ hỗ trợ `vi-VN` cùng `en-US`. Không có bean này thì máy chủ locale `en` sẽ trả nhãn tiếng Anh cho request không gửi `Accept-Language`.

---

## 9. Response

`@ResponseWrapper` trên controller đưa toàn bộ `SearchVouchersResponse` vào `data` của `ResponseTemplate`:

```jsonc
{
  "status": 200, "code": "...", "success": true, "message": "...", "timestamp": "...",
  "data": {
    "keyword": "...",            // echo
    "serviceCode": null,         // echo, LUÔN serialize kể cả null
    "expireWarningDate": 7,      // = bff.eligible.expire-warning-days
    "defaultTab": "all",
    "selectedTab": "all",
    "tabs": [
      { "code": "all", "label": "Tất cả", "labelI18n": {}, "default": true,
        "order": 1, "count": 42, "sort": [{"field":"endDate","direction":"ASC"}] },
      { "code": "expiring_soon", "label": "Sắp hết hạn", "default": false, "order": 2,
        "count": 3,
        "filter": { "conditions": [
          {"field":"endDate","operator":"LTE","value":"7","unit":"DAYS"},
          {"field":"endDate","operator":"GTE","value":"0","unit":"DAYS"},
          {"field":"status","operator":"IN","value":"ACTIVE,AVAILABLE,PARTIALLY_USED"},
          {"field":"campaignStatus","operator":"NOT_IN","value":"INITIALIZING,..."} ] },
        "sort": [{"field":"endDate","direction":"ASC"}] }
    ],
    "content": [ /* OfferItem[], cùng khuôn với data của API #12 */ ],
    "pageable": {}, "totalElements": 42, "totalPages": 5,
    "first": true, "last": false, "size": 10, "number": 0,
    "numberOfElements": 10, "empty": false, "sort": {}
  }
}
```

`tabs[].filter` được echo từ **chính** `TabCondition` đang thực thi (`toFilterCondition()`), nên mô tả gửi client không thể lệch với query.

---

## 10. Bảng lỗi

| Tình huống | Nơi phát sinh | HTTP | `code` |
|---|---|---|---|
| Thiếu, sai tiền tố, hoặc rỗng header `Authorization` | `VtmAuthFilter` | 401 | `UNAUTHORIZED` |
| Token sai chữ ký, hết hạn, sai định dạng, sai issuer | `VtmTokenVerifier` | 401 | `UNAUTHORIZED` |
| `tab` ngoài danh mục, `size` lớn hơn 100, `page` âm, `keyword` quá 255 ký tự | `InvalidParamsExceptionHandler` | 400 | `INVALID_PARAMS` kèm `errors[]` |
| Token hợp lệ nhưng claim `usr` rỗng | `VtmVoucherService.resolveCustomerId` | 422 | `PP-BFF-VTM-2006` |
| Khách chưa có voucher nào | — | 200 | `content: []`, `empty: true`, badge bằng 0 |
| Endpoint không tồn tại | `EndpointNotFoundExceptionHandler` | 404 | — |

**Không có nhánh `CUSTOMER_NOT_FOUND`** ở #13: `customer_voucher_view` là kho voucher, không phải danh bạ khách. Danh tính đã do filter xác thực, không có dòng nào nghĩa là ví rỗng.

**Không có rate limit** trên đường này: không filter throttle, `@RateLimiter` của promix chưa có aspect. Trần đồng thời thực tế chỉ là Hikari pool 20 kết nối.

---

## 11. Hai nhánh thay thế (không chạy với cấu hình hiện tại)

### `VoucherApiAdapter` khi `bff.voucher.read-model.enabled=false`

Gọi `CustomerVoucherFeignClient.listVouchers(customerId, "available", null, page, size)` sang **pp-customer** (`voucher_warehouse`), rồi lọc keyword và tab **trong bộ nhớ, chỉ trên trang đã lấy về**. Hệ quả đã biết:
- `totalElements` lấy từ downstream (chưa lọc) nhưng `content` đã lọc, tổng và nội dung không khớp.
- Badge `expiring_soon` đếm trên trang hiện tại, badge `all` là tổng downstream.
- Keyword khớp một phần trên `title`, `brand.name`, `content`, `description`, **khác** hẳn read model (mã voucher khớp tuyệt đối, khớp trên cột normalized).
- Dùng `VoucherSearchSupport` với tab hard-code, **không** đọc `VoucherTabCatalog`, nên cấu hình tab không có tác dụng ở nhánh này.
- Feign lỗi thì nuốt và trả trang rỗng.

### `MockVoucherApiClient` khi `bff.mock.enabled=true`
Dữ liệu tĩnh cho local dev.

---

## 12. Read model đến từ đâu

VIEW `customer_voucher_view` (changeset `020-validation-order-bounds.xml`):

```sql
FROM voucher_ownership o
LEFT JOIN campaign_offer  ofr ON ofr.campaign_id    = o.campaign_id
LEFT JOIN coupon_display  d   ON d.coupon_config_id = o.coupon_config_id
LEFT JOIN validation_rule vr  ON vr.rule_id         = ofr.validation_rule_id
```

Cột quan trọng cho #13: `title_normalized` và `merchant_name_normalized` (từ `coupon_display`), `expiration_date = COALESCE(o.expiration_date, ofr.expiration_date)`, `campaign_status`, `applicable_service_codes`, `excluded_service_codes`, `applies_to_all` (từ `campaign_offer`), `status`, `reserved_until`, `redeemed_at`, `created_at` (từ `voucher_ownership`).

Bốn bảng gốc được nuôi bằng Kafka projection, group `promotion-vtm-bff`:

```mermaid
flowchart LR
    PC["pp-customer<br/>CustomerVoucher Added Redeemed Revoked"] -->|"topic customer-event"| VPC["VoucherProjectionConsumer"]
    CP["pp-coupon<br/>VoucherEvent"] -->|"promotion_voucher_event"| VSC["VoucherStatusProjectionConsumer"]
    CC["pp-coupon<br/>CouponConfigEvent"] -->|"promotion_coupon_config_event"| CDC["CouponDisplayProjectionConsumer"]
    CM["pp-campaign<br/>CampaignEvent"] -->|"promotion_campaign_event"| CPC["CampaignProjectionConsumer"]
    PE["pp-pricing-engine<br/>DiscountEvent"] -->|"promotion_discount_event"| ODC["OfferDiscountProjectionConsumer"]
    VL["pp-validation<br/>ValidationEvent"] -->|"promotion_validation_event"| VRC["ValidationRuleProjectionConsumer"]

    VPC --> T1[("voucher_ownership")]
    VSC --> T1
    CDC --> T2[("coupon_display")]
    CPC --> T3[("campaign_offer")]
    ODC --> T3
    VRC --> T4[("validation_rule")]

    T1 --> VIEW[("customer_voucher_view")]
    T2 --> VIEW
    T3 --> VIEW
    T4 --> VIEW
    VIEW --> API["API 13"]
```

Nghĩa là **độ trễ và độ đầy đủ của #13 phụ thuộc hoàn toàn vào projection**, không phụ thuộc downstream lúc đọc. Một chiến dịch vừa đổi trạng thái mà event chưa tới thì `campaign_status` vẫn là giá trị cũ và voucher vẫn hiện theo giá trị cũ.

---

## 13. Module liên quan, trạng thái clone

Toàn bộ module cần để hiểu luồng #13 **đã có sẵn trong workspace**, không cần clone thêm:

| Vai trò trong luồng #13 | Repo | Có sẵn |
|---|---|---|
| BFF, chủ thể của API | `p2_promotion-vtm-bff` | có |
| Starter `@ResponseWrapper`, `ResponseTemplate`, `PlatformExceptionHandler`, `BusinessRuleException`, cache Redisson | `p2_promotion-promix-platform` | có |
| Hợp đồng event cho projection | `p2_schema-service` | có |
| Nguồn `voucher_ownership`, và nhánh Feign dự phòng `voucher_warehouse` | `p2_promotion-customer` | có |
| Nguồn `coupon_display` và trạng thái mã voucher | `p2_promotion-coupon` | có |
| Nguồn `campaign_offer` | `p2_promotion-campaign` | có |
| Nguồn `discount_*` của `campaign_offer` | `p2_promotion-discount` | có |
| Nguồn `validation_rule` | `p2_promotion-validation` | có |
| `ServiceCatalogPort`, bung `applicableProducts` cho voucher `applies_to_all` | `p2_promotion-product` | có |

Không nằm trong luồng #13, chỉ dùng cho API #01, #03, #12: `p2_promotion-redemption`.

**Thứ thực sự thiếu không phải repo Java mà là cấu hình ngoài repo:**

1. **`app.kafka.topics.customer-event` không được khai ở bất kỳ file properties nào trong repo**, trong khi `VoucherProjectionConsumer` dùng nó làm `topics`. Giá trị này phải đến từ Spring Cloud Config hoặc biến môi trường bên ngoài, thiếu nó thì consumer không resolve được placeholder. Cần file cấu hình của config server hoặc manifest Swarm/K8s để đối chiếu.
2. **Định nghĩa SQL thật của VIEW trên môi trường**: `spring.liquibase.enabled=false` nên changelog trong repo chỉ là mô tả, không phải thứ đang chạy. Muốn chắc chắn thì chạy `SHOW CREATE VIEW customer_voucher_view` trên schema `promotion_vtm_bff`.
3. **Spec nghiệp vụ `13-search-customer-vouchers.md`** được trích dẫn khắp code nhưng không có trong `docs/`.

---

## 14. Quan sát trong lúc đọc code

- **Số câu truy vấn tăng tuyến tính theo số tab**: mỗi tab cấu hình thêm là thêm một câu `COUNT` trên toàn tập của khách, mỗi request. Bộ mặc định 2 tab đã là 4 round-trip.
- **`@CircuitBreaker` và `@Retry` của resilience4j trong `VoucherApiAdapter` không có aspect**: repo chỉ kéo `resilience4j-annotations`, thiếu `resilience4j-spring-boot3`, nên block `resilience4j.*` trong `application.properties` không được bind. Điều này chỉ ảnh hưởng nhánh Feign, nhánh đang chạy không dùng tới.
- **`LIKE '%kw%'` trên `title_normalized` và `merchant_name_normalized`** không dùng được index tiền tố. Với `customer_source_id` đã lọc trước thì tập quét nhỏ, nhưng đây là điểm cần theo dõi khi một khách giữ nhiều voucher.
- **`groupRank` ở SQL và cặp `VoucherDisplayOrder.groupOf` cộng `VoucherStatusLabelSupport.effectiveStatus` ở Java là hai bản sao của cùng một luật.** Javadoc đã cảnh báo lệch một vế thì thứ tự SQL không khớp trạng thái khách nhìn thấy, mọi thay đổi phải sửa đồng thời hai nơi.
- **Javadoc của `VoucherTabCatalog` nói ngược với code**: phần mở đầu khẳng định cấu hình sai làm service **không start**, nhưng constructor bọc `validate(...)` trong `try/catch (Exception)` và lùi về bộ tab mặc định kèm `log.warn`. Hành vi thật là fail-soft, không fail-fast, nên một cấu hình tab sai sẽ đi vào production dưới dạng thanh tab bỗng quay về mặc định chứ không phải một lần start hỏng.
