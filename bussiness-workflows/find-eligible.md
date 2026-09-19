# Phân tích luồng chạy & nguyên nhân chậm — API #01 Find Eligible Campaigns

| | |
|---|---|
| **Endpoint client gọi** | `POST {context-path}/api/v1/vtm/redemptions/eligible` (pp-bff-vtm) |
| **Endpoint downstream** | `POST {context-path}/v1/redemption/eligible` (pp-redemption) |
| **Entry point** | `VtmRedemptionController.findEligibleCampaigns()` |
| **Ngày phân tích** | 15/09/2026 |
| **Phạm vi đọc code** | `p2_promotion-vtm-bff`, `p2_promotion-redemption` (repo `pp-rule-engine`, `pp-pricing-engine`, `pp-coupon`, `pp-segment`, `pp-customer`, `pp-product` **không có trong workspace** — phần đó suy luận từ hợp đồng gọi + số đo ghi trong comment code) |

---

# PHẦN 1 — DÀNH CHO BA / PM / PO

## 1.1. API này đang làm gì (bằng ngôn ngữ nghiệp vụ)

Khi người dùng mở màn hình "Ưu đãi" trong app VTM, app gọi **một** API duy nhất và mong nhận về **hai danh sách**:

- **"Ưu đãi của tôi"** (`myOffers`) — những mã khách đã sở hữu và còn dùng được cho đơn hàng này.
- **"Ưu đãi khác"** (`otherOffers`) — những chiến dịch khách **chưa** có mã nhưng vẫn đủ điều kiện nhận/dùng.

Để trả lời được, hệ thống phải làm 3 việc lớn cho **từng chiến dịch đang chạy**:

1. **Hỏi "khách này có đủ điều kiện không?"** → máy quy tắc (rule engine) chạy luật nghiệp vụ.
2. **Hỏi "giảm được bao nhiêu tiền?"** → máy tính giá (pricing engine) chạy công thức giảm giá.
3. **Hỏi "còn lượt không, còn hạn không, có bị trùng/xung đột với ưu đãi khác không?"** → kho mã, chính sách gộp ưu đãi.

## 1.2. Vì sao chậm — giải thích không dùng thuật ngữ kỹ thuật

> **Nguyên nhân gốc: hệ thống đang "tính toàn bộ để trả về 20 dòng".**

Hãy hình dung một siêu thị có **350 chương trình khuyến mãi**. Khách vào quầy, app chỉ cần hiển thị **20 ưu đãi đầu tiên**. Nhưng hiện tại hệ thống làm thế này:

1. Lấy **cả 350** chương trình ra.
2. Chạy luật nghiệp vụ cho **cả 350**.
3. Tính số tiền giảm cho **cả 350** (chia lô 50 chương trình/lần, **làm tuần tự**, 7 lượt liên tiếp).
4. Chạy lại luật nghiệp vụ **lần hai** cho **cả 350** (vì hai tầng quy tắc khác nhau, mỗi chương trình phải hỏi 2 lần → **700 lượt hỏi**).
5. Hỏi kho mã xem còn lượt không cho **cả 350**.
6. **Sắp xếp, rồi mới cắt ra 20 dòng để trả về.**

→ **330 chương trình bị tính toán đầy đủ rồi vứt đi.**

### Ba "hệ số nhân" làm cho việc lãng phí trên bị lặp lại nhiều lần

| Hệ số nhân | Mô tả nghiệp vụ | Hệ quả |
|---|---|---|
| **×2 — hai danh sách** | "Ưu đãi của tôi" và "Ưu đãi khác" là **hai lần gọi độc lập** xuống lõi xử lý. May mắn là chúng chạy **song song** nên không cộng dồn thời gian, nhưng **gấp đôi tải** lên toàn bộ hệ thống phía sau. |
| **×N — lọc sau khi lấy** | BFF lấy dữ liệu về rồi mới **lọc bỏ** (hết hạn, đã sở hữu, chiến dịch tạm dừng…). Lọc xong không đủ 20 dòng → **gọi lại lần nữa** để lấy thêm. Mỗi lần gọi lại là **lặp trọn vẹn 6 bước ở trên**. |
| **×5 — khi người dùng gõ tìm kiếm** | Nếu request có **từ khoá tìm kiếm** hoặc **lọc theo loại giảm giá**, hệ thống bỏ hẳn cơ chế phân trang và **gom tối đa 500 bản ghi** cho mỗi danh sách → **tối đa 5 lượt gọi liên tiếp mỗi danh sách**. |
| **+1 — lượt gọi bù** | Danh sách "Ưu đãi khác" phải gọi **thêm một lượt nữa** (tối đa 200 chiến dịch) để vớt các chiến dịch chưa gắn quy tắc — máy quy tắc không bao giờ tự liệt kê chúng. |

### Con số đo thật đã được ghi lại trong code

Đây là số **do chính đội dev đo và ghi chú lại trong mã nguồn**, không phải ước lượng của tài liệu này:

| Số đo | Nguồn |
|---|---|
| **~17 giây / 1 lượt gọi** pp-redemption | `VtmRedemptionService` (đo 04/08/2026) |
| **~53 giây** cho lượt gọi của nhóm "Ưu đãi khác" | `VtmRedemptionService` (đo local) |
| **~40 giây** riêng cho bước chạy luật lần hai (700 lượt hỏi) khi bộ nhớ đệm còn nguội | `UnifiedRuleEngineAdapter` (đo 10/08/2026) |
| **~7,5 giây** cho cùng bước đó khi bộ nhớ đệm đã ấm | `UnifiedRuleEngineAdapter` |
| **2.103 ms** chỉ để tra công thức giảm giá của 95 chiến dịch (đã được sửa bằng gọi theo lô) | `PricingEngineServiceClient` |
| **353 chiến dịch → 706 lượt hỏi luật** trong một request | `UnifiedRuleEngineAdapter` |

> ⚠️ **Chênh lệch 40s (nguội) vs 7,5s (ấm) là rủi ro nghiệp vụ, không chỉ là rủi ro hiệu năng.** Ở lượt nguội, lời gọi bị quá hạn → hệ thống **fail-open** (bỏ qua kiểm tra và cho tất cả đi qua). Đo 10/08/2026: đáng lẽ phải ẩn 171 chiến dịch thì ẩn 0. **Nghĩa là: khi hệ thống chậm, kết quả trả về cũng có thể SAI** — hiển thị ưu đãi mà khách không đủ điều kiện.

## 1.3. Thời gian phản hồi ước tính theo kịch bản

| Kịch bản người dùng | Số lượt gọi lõi | Thời gian ước tính |
|---|---|---|
| Mở màn hình lần đầu, dữ liệu sạch, đủ 20 dòng ngay lượt đầu | 2 (song song) + 1 bù | **~17–25 s** |
| Mở màn hình, nhưng phần lớn bản ghi bị lọc bỏ → phải gọi thêm trang | 4–6 | **~35–70 s** |
| **Người dùng gõ từ khoá tìm kiếm** | tới 10 (5/nhóm) | **~60–90 s** |
| Bộ nhớ đệm của máy quy tắc còn nguội (sau khi deploy / sau giờ thấp điểm) | — | **cộng thêm 30–40 s**, và **kết quả có nguy cơ sai** |

*(Các con số trên là suy ra từ số đo ghi trong code × số lượt gọi mà logic hiện tại sinh ra. Cần đo lại trực tiếp trên môi trường thật để chốt — xem mục 2.6.)*

## 1.4. Điều BA/PM/PO cần quyết định

Đây là những quyết định **nghiệp vụ**, dev không tự quyết được, và chúng là điều kiện tiên quyết để sửa triệt để:

| # | Câu hỏi cần chốt | Vì sao quan trọng |
|---|---|---|
| **Q1** | **Có chấp nhận giới hạn số ưu đãi hiển thị không?** VD: chỉ chào tối đa 50 chiến dịch "Ưu đãi khác" tốt nhất thay vì toàn bộ 350. | Đây là đòn bẩy mạnh nhất. Giảm 350→50 cắt ~85% khối lượng tính toán. |
| **Q2** | **Số tiền giảm ước tính có bắt buộc phải hiện ngay ở màn hình danh sách không?** Hay có thể chỉ hiện khi khách bấm vào từng ưu đãi? | Bước tính giá đang chạy cho **toàn bộ** ứng viên. Nếu chỉ cần khi xem chi tiết, bỏ được một chuỗi gọi tuần tự nặng. Lưu ý: hiện nay **không bỏ được hoàn toàn**, vì có luật nghiệp vụ dạng "tổng giá trị **sau giảm giá** ≥ X" cần biết số tiền giảm mới đánh giá được (PROM-1427). Cần chốt: **có còn dùng loại điều kiện đó không?** |
| **Q3** | **Tìm kiếm theo từ khoá phải quét toàn bộ ưu đãi, hay chỉ quét trong danh sách đang hiển thị?** | Đang quét toàn bộ (gom 500 bản ghi) → đây là kịch bản chậm nhất. Nếu chấp nhận tìm trong phạm vi hẹp hơn, hoặc chuyển tìm kiếm xuống tầng dữ liệu, tiết kiệm rất lớn. |
| **Q4** | **Dữ liệu ưu đãi được phép "cũ" bao lâu?** 30 giây? 5 phút? | Quyết định việc có được dùng bộ nhớ đệm hay không. Hiện tại **không có lớp đệm nào** cho kết quả này — mỗi lần kéo-để-làm-mới là tính lại từ đầu 100%. |
| **Q5** | **Mức SLA mong muốn cho màn hình này là bao nhiêu?** | Không có con số mục tiêu thì không biết dừng tối ưu ở đâu. Gợi ý tham chiếu: p95 < 3 s cho màn hình danh sách trên mobile. |
| **Q6** | **Khi hệ thống quá tải, ưu tiên "trả chậm nhưng đúng" hay "trả nhanh nhưng có thể thiếu/sai"?** | Hiện tại hệ thống đang **âm thầm chọn "nhanh nhưng sai"** (fail-open) mà không ai quyết định điều đó. Cần một quyết định có chủ đích. |

## 1.5. Điều KHÔNG phải nguyên nhân (để khỏi đi nhầm hướng)

- ❌ **Không phải do xác thực/đăng nhập** — token được kiểm tra cục bộ, không gọi mạng.
- ❌ **Không phải do cơ sở dữ liệu của BFF** — các truy vấn đều có chỉ mục, chạy ở mức mili-giây, không đáng kể so với 17 s.
- ❌ **Không phải do lỗi N+1 kinh điển** — đội dev đã sửa (gọi theo lô). Vấn đề còn lại là **quy mô tập dữ liệu** và **chuỗi gọi tuần tự**, không phải số lượt gọi lẻ tẻ.
- ❌ **Không phải do mạng chậm** — các dịch vụ nằm cùng cụm.

---
---

# PHẦN 2 — DÀNH CHO DEV

## 2.1. Sơ đồ luồng gọi đầy đủ

```
MOBILE VTM
  │ POST {ctx}/api/v1/vtm/redemptions/eligible
  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ pp-bff-vtm                                                               │
│ VtmRedemptionController.findEligibleCampaigns()                          │
│   └─ VtmRedemptionService.findEligibleCampaigns()          [service:130] │
│                                                                          │
│  ① readModel.loadOwned(msisdn)          → SELECT customer_voucher_view    │
│  ② readModel.loadOwnedDisplay(msisdn)   → SELECT customer_voucher_view    │
│       ⚠️ ① và ② chạy CÙNG MỘT CÂU SQL hai lần (xem 2.3-B)                 │
│                                                                          │
│  ③ recordsNeeded() → myNeeded / otherNeeded                              │
│       nếu có keyword || discountTypes  ⇒ needed = 500 (MAX_FETCH_RECORDS) │
│       ngược lại                        ⇒ needed = (page+1)*size + 1      │
│                                                                          │
│  ④ CompletableFuture song song trên "eligibleExecutor" (core=4,max=8,q=50)│
│     ├── callMy()    : filter = campaignId.$in(activeCampaignIds)         │
│     │                 codeHolder = true                                  │
│     └── callOther() : filter = campaignId.$not_in(allOwnedCampaignIds)   │
│                       codeHolder = false                                 │
│                       + gatherCampaignsWithoutRule() ← LƯỢT GỌI THỨ 2    │
│                         (loadClaimableCampaignIds → $in ≤200 id)         │
│                                                                          │
│  mỗi nhánh: gatherEligible() = VÒNG LẶP while(chưa đủ needed)            │
│     page=0,1,2,…  size = min(100, 500-fetched)                           │
│     ⇒ tới 5 lượt gọi pp-redemption / nhánh                               │
│     mỗi vòng (nhóm OTHER) còn +3 truy vấn DB:                            │
│       loadDisplayByCampaignIds / loadDisabledCampaignIds /               │
│       loadCampaignExpiryByCampaignIds                                    │
│                                                                          │
│  ⑤ appendExpiredOwned → toPageData (lọc discountTypes/keyword, sort, cắt) │
└──────────────────────────────────────────────────────────────────────────┘
  │ RedemptionApiAdapter.findEligible()  @CircuitBreaker(name="redemption")
  │ Feign → POST {ctx}/v1/redemption/eligible   [read-timeout 60s, hc5 60s]
  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ pp-redemption — FindEligibleCampaignsService.findEligibleCampaigns()     │
│                                                                          │
│  Stage 0  customerResolutionPort.resolve()      → pp-customer      (HTTP)│
│  Stage 0  productResolutionPort.resolveBySku()  → pp-product       (HTTP)│
│  Stage 0  resolveProductsFailOpen()             → pp-product       (HTTP)│
│  Stage 0  fetchCustomerSegments()               → pp-segment       (HTTP)│
│                                                                          │
│  Step 2   campaignEligibilityPort.checkEligibility()                     │
│             → pp-rule-engine POST /v1/discovery/eligible           (HTTP)│
│             ⇒ TRẢ VỀ TOÀN BỘ ỨNG VIÊN, KHÔNG PHÂN TRANG                  │
│           addCampaignsWithoutRule()  (chỉ khi filter dùng $in)           │
│                                                                          │
│  Step 3   gating (in-memory)                                             │
│  Step 3b  typeAvailabilityPort.checkAvailabilityBatch()                  │
│             → pp-coupon, CHIA LÔ TUẦN TỰ                           (HTTP)│
│  Step 3b2 enrichDiscounts()  ⚠️ CHẠY TRÊN TOÀN BỘ ỨNG VIÊN               │
│             → pp-pricing-engine /calculate-batch, LÔ 50, TUẦN TỰ   (HTTP)│
│             → pp-pricing-engine /discount-formulas/by-campaigns    (HTTP)│
│  Step 3c  applyValidationRules()  ⚠️ CHẠY TRÊN TOÀN BỘ ỨNG VIÊN          │
│             → pp-rule-engine POST /v1/batch-evaluate                     │
│               2 subject/campaign, LÔ 200 subject, TUẦN TỰ          (HTTP)│
│  Step 4   stackingRulePort.analyzeStacking()                       (HTTP)│
│                                                                          │
│  Step 5   sort + PHÂN TRANG  ⬅️ CẮT TRANG Ở ĐÂY, SAU TẤT CẢ              │
│  Step 7   toCampaignItem() cho đúng trang                                │
└──────────────────────────────────────────────────────────────────────────┘
```

## 2.2. Bảng phóng đại lời gọi (call amplification)

Giả thiết: `C` = số chiến dịch ứng viên (đo thật: **353**), `P` = số lượt gọi pp-redemption mà một nhánh BFF thực hiện.

| Tầng | Công thức | Với C=353, P=1 | Với C=353, P=5 (có keyword) |
|---|---|---|---|
| BFF → pp-redemption | `2 nhánh × P + 1 lượt bù` | 3 | 11 |
| pp-redemption → pp-rule-engine (discovery) | `1 / lượt` | 3 | 11 |
| pp-redemption → pp-rule-engine (batch-evaluate) | `⌈2C/200⌉ / lượt` = 4 | **12** | **44** |
| pp-redemption → pp-pricing-engine (calculate) | `⌈C/50⌉ / lượt` = 8 | **24** | **88** |
| pp-redemption → pp-pricing-engine (formula) | `⌈C/50⌉ / lượt` = 8 | **24** | **88** |
| pp-redemption → pp-coupon (usage) | `⌈C/MAX⌉ / lượt` | ~3–24 | ~11–88 |
| pp-redemption → pp-customer / pp-product ×3 / pp-segment | `4 / lượt` | 12 | 44 |
| **Tổng lời gọi HTTP hạ tầng** | | **≈ 80–100** | **≈ 290–370** |

→ **Một lần chạm màn hình của một người dùng sinh ra ~100 lời gọi HTTP nội bộ, phần lớn là tuần tự.**

## 2.3. Danh sách nút thắt, xếp theo mức tác động

### 🔴 A — Không đẩy phân trang xuống (nghiêm trọng nhất)
**Vị trí:** `FindEligibleCampaignsService.java` Step 3b2 → Step 5.
`command.pagination()` chỉ được dùng ở **Step 5**, sau khi `enrichDiscounts` (Step 3b2) và `applyValidationRules` (Step 3c) đã chạy trên **toàn bộ** `gatedCampaigns`.

Điều này **không phải sơ suất** — comment ở dòng ~230 ghi rõ PROM-1427 đã cố ý kéo pricing lên trước rule gate, vì điều kiện `order.total.*` ("Tổng giá trị sau giảm giá") cần biết số tiền giảm mới đánh giá được. **Đây là ràng buộc nghiệp vụ, không gỡ được bằng refactor thuần kỹ thuật** → phụ thuộc câu hỏi **Q2** ở Phần 1.

### 🔴 B — Chuỗi gọi tuần tự có thể song song hoá
Ba chỗ chia lô rồi gọi **tuần tự** trong vòng `for`:
- `UnifiedRuleEngineAdapter.evaluateInChunks()` — lô 200 subject (`redemption.rule-engine.batch.max-subjects-per-call`, mặc định 200).
- `DiscountCalculationAdapter.calculateDiscountsBatch()` — `MAX_BATCH_SIZE = 50`, vòng `for` tuần tự.
- `CampaignUsageAvailabilityAdapter.fetchConfigs()` — vòng `for` tuần tự.

Ngoài ra Stage 0 có **4 lời gọi HTTP độc lập nhau** (customer, sku, product, segment) chạy **nối tiếp** — đây là phần dễ song song hoá nhất, không đụng nghiệp vụ.

> Lưu ý phản biện: comment trong `UnifiedRuleEngineAdapter` ghi "chia lô 200 hết 7.5s, một lời gọi duy nhất hết 7.4s" ⇒ trên đường **ấm**, chia lô không gây thêm độ trễ, tức nút thắt nằm ở **bản thân rule-engine**, không ở việc chia lô. Song song hoá ở đây sẽ giúp chủ yếu ở **đường nguội**.

### 🟠 C — Truy vấn read model bị lặp nguyên văn
`EligibleReadModelAdapter.loadOwned()` và `loadOwnedDisplay()` **cùng gọi** `repository.findByCustomerSourceIdAndGroupType(customerSourceId, "MY")`. Trong `findEligibleCampaigns()` cả hai đều được gọi (dòng ~152 và ~158) ⇒ **chạy y hệt một câu SQL trên `customer_voucher_view` hai lần**.

`customer_voucher_view` là **SQL VIEW 4 bảng** (`voucher_ownership` ⟕ `campaign_offer` ⟕ `coupon_display` ⟕ `validation_rule`, xem `019-campaign-start-date.xml`). Chi phí không lớn so với 17 s nhưng là món **sửa dễ, không rủi ro** — comment trong `toOwnedRow()` đã tự thừa nhận "hai nguồn đọc CÙNG một truy vấn".

### 🟠 D — Vòng lặp `gatherEligible` nhân lượt gọi
`VtmRedemptionService.gatherEligible()` lặp tới khi đủ `needed` **bản ghi đã hậu-lọc**. Mỗi vòng = **một lượt pp-redemption ~17 s**. Đã có 5 điều kiện dừng sớm (trang rỗng / thiếu / cả trang bị lọc sạch / tới totalPages / chạm trần 500), nhưng khi tỉ lệ lọc bỏ cao (~50%) thì vẫn phải 2–3 vòng.

Cộng thêm `gatherCampaignsWithoutRule()` — **một lượt gọi nữa** cho nhóm OTHER. Comment ghi lượt này "rẻ" (145 ms local vì discovery không khớp ứng viên nào), nhưng nó vẫn phải chạy trọn `enrichDiscounts` + `applyValidationRules` cho tới 200 campaign được bù vào ⇒ **cần đo lại trên staging trước khi coi là rẻ**.

### 🟠 E — `bffSideSearch` vô hiệu hoá phân trang
`EligibleFilterSupport.runsBffSideSearch(keyword, filterOptions)` → `recordsNeeded()` trả thẳng `MAX_FETCH_RECORDS = 500`. Với `DOWNSTREAM_MAX_PAGE_SIZE = 100` ⇒ **tới 5 lượt gọi tuần tự/nhánh**. Đây là kịch bản chậm nhất và nó xảy ra **chính lúc người dùng đang tương tác** (gõ tìm kiếm).

Nguyên nhân kiến trúc: `keyword` và `discountTypes` được lọc **tại BFF** (sau khi ghép display) chứ không đẩy xuống DSL của downstream.

### 🟡 F — Cấu hình timeout theo từng downstream đang bị vô hiệu (cần xác minh)
`ValidationEngineFeignConfig` khai `Request.Options(connect=5s, read=10s)` qua `@Bean`. Nhưng `application.properties` lại đặt:
```properties
spring.cloud.openfeign.client.config.default.read-timeout=60000
```
Spring Cloud OpenFeign mặc định `spring.cloud.openfeign.client.default-to-properties=true` (**đã kiểm tra: không repo nào override**) ⇒ **cấu hình từ properties được áp SAU và ghi đè `@Bean` Java config**.

⇒ Giả thuyết: mọi giá trị `redemption.external-services.*.read-timeout` (5 s / 10 s) là **dead config**, timeout thực tế của **mọi** downstream pp-redemption là **60 s**.

**Bằng chứng gián tiếp ủng hộ giả thuyết này:** comment trong `UnifiedRuleEngineAdapter` ghi *"vượt qua read-timeout 60s ⇒ Feign ném Read timed out"* — nếu `Request.Options` 10 s có hiệu lực thì lời gọi đã phải chết ở 10 s, không phải 60 s.

> ⚠️ **Đây là suy luận từ hành vi framework, chưa xác minh trực tiếp.** Cách kiểm chứng: gọi `GET /actuator/configprops` hoặc log `feign.Request.Options` lúc khởi động. **Cần xác nhận trước khi sửa.**

### 🟡 G — Thread pool `eligibleExecutor` dễ nghẽn khi có tải
`BffAsyncConfig`: `core=4, max=8, queue=50`.
`ThreadPoolTaskExecutor` (kế thừa `ThreadPoolExecutor`) **chỉ tạo thêm thread vượt core khi hàng đợi ĐẦY**. Với queue=50 và mỗi task ~17–53 s:
- 2 người dùng đồng thời = 4 task = vừa đủ core.
- Người thứ 3 trở đi → task vào **hàng đợi**, không kích hoạt thread mới cho tới khi đủ 50 task xếp hàng.
- Task thứ 51 → `AbortPolicy` mặc định → `RejectedExecutionException` → **500**.

⇒ Với ~2 request đồng thời, thời gian chờ đã bắt đầu cộng dồn; hiệu năng sụp đổ phi tuyến. **Cần kiểm chứng bằng metric `executor.queued`/`executor.active`.**

### 🟡 H — Không có lớp cache nào cho kết quả
Không tìm thấy `@Cacheable` hay lớp đệm nào trên đường `findEligible`. pp-redemption chỉ có cache cho **công thức giảm giá** (`promix.cache.redisson.caches.discountFormulaByCampaign.ttl=5m`). Mỗi lần kéo-để-làm-mới = tính lại 100%.

Đáng chú ý: **tập ứng viên của nhóm "Ưu đãi khác" gần như giống nhau giữa các khách hàng** (chỉ khác phần `$not_in` và facts của khách) — đây là ứng viên tốt cho cache theo `(orderFingerprint, segmentSet)`.

### 🟢 I — Rủi ro đúng đắn do timeout (không phải hiệu năng, nhưng phát hiện cùng lúc)
Chuỗi: `batch-evaluate` timeout → `UnifiedRuleEngineAdapter` gán `EVALUATION_ERROR` → mã này nằm trong `RuleReasonCodePolicy.NOT_A_RULE_VIOLATION` → `classifyCampaignByRule()` cho **đi qua hết** → fail-open toàn bộ.

Cùng một pattern fail-open lặp lại ở **6 chỗ**: `resolveProductsFailOpen`, `fetchCustomerSegments`, `filterOutUsageExhausted`, `applyValidationRules`, `calculateDiscountsBatch`, `loadDisabledCampaignIds`. Mỗi chỗ đều hợp lý **riêng lẻ**, nhưng cộng lại thì **khi hệ thống chậm, gần như mọi cổng kiểm soát đều mở**. Đo 10/08/2026: đáng lẽ ẩn 171 campaign → ẩn 0.

## 2.4. Đề xuất khắc phục, xếp theo tỉ lệ lợi ích/chi phí

### Nhóm 1 — Sửa được ngay, không đụng nghiệp vụ, rủi ro thấp

| # | Việc | Vị trí | Lợi ích ước tính |
|---|---|---|---|
| 1.1 | Song song hoá Stage 0 (customer / sku / product / segment) bằng `CompletableFuture` | `FindEligibleCampaignsService` dòng ~115–165 | −300…800 ms/lượt |
| 1.2 | Gộp `loadOwned` + `loadOwnedDisplay` thành **một** truy vấn, dựng cả hai kết quả từ cùng danh sách row | `EligibleReadModelAdapter` | −1 truy vấn view 4-bảng/request |
| 1.3 | Song song hoá các vòng chia lô (`evaluateInChunks`, `calculateDiscountsBatch`, `fetchConfigs`) trên executor có giới hạn | 3 adapter | −40…60 % ở đường nguội |
| 1.4 | **Xác minh** giả thuyết (F), rồi chuyển timeout per-client sang `spring.cloud.openfeign.client.config.<tên-client>.*` | `application.properties` | Chặn được hiệu ứng timeout dây chuyền |
| 1.5 | Đổi `eligibleExecutor` sang queue nhỏ (VD 8) + `CallerRunsPolicy`, hoặc nâng core | `BffAsyncConfig` | Hết nghẽn queue khi có tải |

### Nhóm 2 — Cần BA/PM/PO chốt trước (xem Q1–Q4)

| # | Việc | Phụ thuộc |
|---|---|---|
| 2.1 | Đẩy `keyword` / `discountTypes` xuống DSL của downstream để bỏ `bffSideSearch = 500` | **Q3** |
| 2.2 | Trần cứng số ứng viên "Ưu đãi khác" (VD 50) thay vì toàn bộ danh mục | **Q1** |
| 2.3 | Tách `enrichDiscounts` thành 2 pha: pha nhẹ (chỉ campaign có rule dạng `order.total.*`) trước rule gate, pha đầy đủ chỉ cho trang trả về | **Q2** |
| 2.4 | Cache kết quả discovery theo `(customerSegments, orderFingerprint, filterHash)` TTL ngắn | **Q4** |

### Nhóm 3 — Kiến trúc (trung hạn)

- **Đẩy phân trang xuống pp-rule-engine discovery.** Hiện `/v1/discovery/eligible` trả toàn bộ tập, không nhận `page/size`. Đây là thay đổi hợp đồng liên service nhưng là cách duy nhất phá vỡ triệt để mô hình "tính 350 để trả 20".
- **Tách đường đọc:** dựng một read model "ưu đãi đủ điều kiện" được cập nhật bất đồng bộ (đã có sẵn hạ tầng Kafka projection trong BFF — xem `adapter/in/messaging/*`), để #01 chỉ đọc DB thay vì fan-out 100 lời gọi HTTP.
- **Quyết định fail-open có chủ đích (mục I):** phân biệt "hạ tầng lỗi" vs "không có vi phạm", và có chính sách rõ ràng — hoặc trả lỗi, hoặc gắn cờ `degraded=true` trong response để client biết.

## 2.5. Cấu hình nên rà lại

| Property | Nơi | Giá trị hiện tại | Ghi chú |
|---|---|---|---|
| `bff.eligible.other-offers.no-rule-max-ids` | bff | `200` | Đặt `0` để **tắt hẳn** lượt gọi bù — dùng để đo đóng góp thật của nó |
| `spring.cloud.openfeign.client.config.default.read-timeout` | bff | `60000` | Có thể đang che timeout per-client |
| `spring.cloud.openfeign.client.config.default.read-timeout` | redemption | `60000` | Nghi ngờ ghi đè `ValidationEngineFeignConfig` (giả thuyết F) |
| `redemption.rule-engine.batch.max-subjects-per-call` | redemption | `200` (mặc định, không khai trong properties) | Đòn bẩy để thử song song hoá |
| `DiscountCalculationAdapter.MAX_BATCH_SIZE` | redemption | `50` (**hằng số cứng**) | Ràng buộc hợp đồng pricing-engine — muốn đổi phải đổi cả hai phía |
| `EligibleOfferPageSupport.MAX_FETCH_RECORDS` | bff | `500` (**hằng số cứng**) | Nên đưa ra cấu hình để chỉnh không cần deploy |
| `resilience4j.retry.instances.redemption.max-attempts` | bff | `2` | ⚠️ Retry một lời gọi 17 s ⇒ **34 s**. Rà lại xem có nên retry endpoint này không |
| `spring.datasource.hikari.maximum-pool-size` | bff | `20` | Đủ, không phải nút thắt |

## 2.6. Cách xác minh — đo trước khi sửa

Code **đã có sẵn** đầy đủ instrumentation. Không cần thêm gì, chỉ cần bật log và đọc.

**Bước 1 — bật DEBUG trên BFF** để thấy thời gian từng lượt gọi downstream:
```properties
logging.level.vn.viettel.vds.promotion.bff.vtm.application.service.VtmRedemptionService=DEBUG
```
→ sinh ra dòng `VTM #01 [<section>] gọi pp-redemption #N page=… size=… → raw=… <ms>ms`.

**Bước 2 — đọc dòng timing INFO** (đã bật sẵn):
```
VTM #01 timing cust=… | readModel: loadOwned=Xms loadOwnedDisplay=Yms
  | nhánh SONG SONG (KHÔNG cộng dồn): my=Ams other=Bms
  (trong đó loadDisplayByCampaignIds=Cms) | tổng=Tms
```

**Bước 3 — đọc dòng tổng kết nhóm** để biết `calls=` (số lượt gọi thật) và `dừng vì …`:
```
VTM #01 [otherOffers] … | fetch: calls=3 raw=300 cần=500 (trang cuối page=2 size=100, dừng vì …)
```

**Bước 4 — đối chiếu với log của pp-redemption** (`#01 [<requestId>]`, INFO, đã bật sẵn):
```
#01 [id] … | resolve=Xms (customer=…ms, product=…ms/… sku)
  | ruleEngine=Ams → N campaign | hết lượt=Bms | rule=Cms: … | enrich=Dms (… pricing: N target/M lô Ems)
  | trang page=… | TỔNG=Zms
```

Bốn dòng này trả lời trực tiếp: **thời gian tiêu ở stage nào**, **bao nhiêu lượt gọi**, và **bao nhiêu campaign bị tính rồi vứt đi** (`ruleEngineCount` vs `contentSize`).

**Bước 5 — thí nghiệm cô lập** (chạy trên staging, mỗi lần đổi 1 biến):

| Thí nghiệm | Cách làm | Câu trả lời thu được |
|---|---|---|
| Lượt bù no-rule tốn bao nhiêu? | đặt `bff.eligible.other-offers.no-rule-max-ids=0` | xác nhận/bác bỏ "lượt này rẻ" |
| Vòng lặp gather tốn bao nhiêu? | gọi với `sectionCode=myOffers` (bỏ hẳn nhánh other) | tách chi phí 2 nhánh |
| `bffSideSearch` tốn bao nhiêu? | gọi cùng payload có và không có `keyword` | định lượng tác động của **Q3** |
| Cache nguội vs ấm? | gọi 2 lần liên tiếp cùng payload | định lượng chênh lệch 40 s vs 7,5 s |

## 2.7. Những điều tài liệu này **chưa** khẳng định được

Nêu rõ để không ai xây quyết định trên nền không chắc:

1. **Không đọc được `pp-rule-engine`, `pp-pricing-engine`, `pp-coupon`, `pp-segment`, `pp-customer`, `pp-product`** — không có trong workspace. `/v1/discovery/eligible` và `/v1/batch-evaluate` là **hộp đen**. Rất có thể nút thắt lớn nhất nằm **bên trong** rule-engine (quét bảng `assignments`, biên dịch Drools, cache bundle), mà từ đây không nhìn thấy được.
2. **Các con số thời gian đều lấy từ comment trong code** (đo 04/08, 10/08, 11/08/2026), **chưa đo lại trên môi trường hiện tại** (15/09/2026). Code đã qua nhiều lần tối ưu kể từ đó (gọi theo lô, chia chunk) → **số thật hôm nay có thể đã tốt hơn đáng kể**. Mục 2.6 là bước bắt buộc trước khi hành động.
3. **Giả thuyết (F) về timeout bị ghi đè chưa được xác minh trực tiếp**, chỉ suy ra từ hành vi mặc định của Spring Cloud OpenFeign cộng với một dòng comment.
4. **Chưa có số liệu tải thật** (RPS, số người dùng đồng thời, phân bố `C` = số campaign/khách). Kết luận về nghẽn thread pool (mục G) là suy luận từ cấu hình, chưa có metric xác nhận.
5. **Không có trace phân tán** (APM/OpenTelemetry) trong phạm vi đọc. Nếu hạ tầng có sẵn, một trace duy nhất sẽ thay thế được toàn bộ mục 2.6.

---

## Phụ lục — Bản đồ file

| File | Vai trò |
|---|---|
| `p2_promotion-vtm-bff/.../adapter/in/web/VtmRedemptionController.java` | Entry point, `POST /api/v1/vtm/redemptions/eligible` |
| `p2_promotion-vtm-bff/.../application/service/VtmRedemptionService.java` | Điều phối 2 nhánh, vòng lặp `gatherEligible`, lượt bù no-rule |
| `p2_promotion-vtm-bff/.../application/support/EligibleOfferPageSupport.java` | `MAX_FETCH_RECORDS=500`, `DOWNSTREAM_MAX_PAGE_SIZE=100`, `recordsNeeded` |
| `p2_promotion-vtm-bff/.../application/support/EligibleFilterSupport.java` | `runsBffSideSearch`, lọc keyword/discountTypes tại BFF |
| `p2_promotion-vtm-bff/.../adapter/out/external/RedemptionFeignClient.java` | **FeignClient `pp-redemption`** → `POST /v1/redemption/eligible` |
| `p2_promotion-vtm-bff/.../adapter/out/external/RedemptionApiAdapter.java` | `@CircuitBreaker("redemption")`, map lỗi downstream |
| `p2_promotion-vtm-bff/.../adapter/out/persistence/EligibleReadModelAdapter.java` | Đọc `customer_voucher_view`, `campaign_offer`, `coupon_display` |
| `p2_promotion-vtm-bff/.../adapter/config/BffAsyncConfig.java` | `eligibleExecutor` (core=4/max=8/queue=50) + truyền context |
| `p2_promotion-vtm-bff/.../adapter/config/FeignConfig.java` | Forward `Authorization`/`X-Tenant-Id`/`X-Request-Id` |
| `p2_promotion-redemption/.../application/usecase/FindEligibleCampaignsService.java` | **Lõi xử lý** — Stage 0 → Step 7 |
| `p2_promotion-redemption/.../adapter/out/external/RuleEngineDiscoveryAdapter.java` | → `POST /v1/discovery/eligible` |
| `p2_promotion-redemption/.../adapter/out/external/UnifiedRuleEngineAdapter.java` | → `POST /v1/batch-evaluate`, chia lô 200 subject **tuần tự** |
| `p2_promotion-redemption/.../adapter/out/external/DiscountCalculationAdapter.java` | → pricing `/calculate-batch`, lô 50 **tuần tự** |
| `p2_promotion-redemption/.../adapter/out/external/CampaignUsageAvailabilityAdapter.java` | → pp-coupon, chia lô **tuần tự** |
| `p2_promotion-redemption/.../adapter/out/external/client/*FeignConfig.java` | Timeout per-client (nghi bị properties ghi đè — mục F) |
