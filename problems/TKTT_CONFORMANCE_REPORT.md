# TKTT Conformance Report — Promotion Platform

> Thực hiện theo `AI_AGENT_CONFIRM_CHECKLIST_TKTT_PRM.md`
> Ngày chạy: 2026-09-13 · Agent: Claude Opus 5

---

## Scope

| Mục | Giá trị |
|---|---|
| **Environment kiểm tra** | `production` (qua `app-config/*/production/application.yml`) + source code trên nhánh mặc định của từng repo |
| **Repositories checked** | 30 git repo dưới `/mnt/data/Data/code/promotion` (23 Java/Maven, 2 frontend, 3 config/doc) |
| **Design baseline dùng thực tế** | `Promotion-Phase-2-Markdown/03-3-tai-lieu-ky-thuat/001-tai-lieu-thiet-ke-tong-the-promotion-platform-phase-2.md` (bản markdown của PDF trang 125–173) |
| **Sizing baseline dùng thực tế** | `.../013-bao-cao-dinh-muc-ha-tang-nen-tang-khuyen-mai.md` (bản markdown của PDF trang 31–63) |
| **Deployed inventory** | `.../04-4-source-code-va-thong-tin-cac-moi-truong/003-1-danh-sach-service-database.md` |
| **Cluster / DB / dashboard access** | **KHÔNG CÓ.** Không có kubectl, không có DB session, không có Grafana/Prometheus/ArgoCD. Mọi kết luận runtime → `NOT_VERIFIABLE` |
| **Không tìm thấy trong estate** | Helm chart, Kubernetes manifest, HPA, NetworkPolicy, PDB, Istio config, backup job, DR runbook, load-test report |

---

## Summary

| Status | Count | Ghi chú |
|---|---:|---|
| PASS | 71 | Có evidence file+symbol/config-key cụ thể |
| FAIL | 43 | Có evidence chứng minh khác/thiếu so với thiết kế |
| PARTIAL | 38 | Đúng một phần |
| NOT_VERIFIABLE | 168 | Chủ yếu §9 perf, §10 K8s, §14 DR, §15 alert threshold, §12.4 security testing |
| NOT_APPLICABLE | 21 | Chủ yếu các check gateway (service không tồn tại trong estate) |
| DESIGN_CONFLICT | 23 | Xem bảng Document Conflicts |
| **Tổng** | **364** | |

> Tỷ lệ `NOT_VERIFIABLE` cao (~46%) là hệ quả trực tiếp của việc chỉ có source+config, không có runtime access. Đây **không** phải kết luận "đạt" hay "không đạt" — đây là khoảng trống evidence phải được đóng trước nghiệm thu.

---

## ⚠️ Phát hiện quan trọng nhất: checklist đang dùng baseline CŨ

**`DOC-000 [P0] — DESIGN_CONFLICT`**

Checklist `AI_AGENT_CONFIRM_CHECKLIST_TKTT_PRM.md` đặt expected values theo một bản sizing **cũ hơn** tài liệu v7 mà nó trích dẫn. Đối chiếu trực tiếp:

| Chỉ số | Checklist kỳ vọng | Tài liệu v7 thực tế (`013-...md`) | Dòng evidence |
|---|---|---|---|
| Transactions/ngày | 1M/day, 365M/năm | **2.33M/day, 840M/năm** (70M/tháng) | `013-...md:355` |
| TPS average | 11.6 TPS | **27 TPS** | `013-...md:361, 402, 711` |
| TPS peak | 116 TPS | **65 TPS** | `013-...md:355, 711` |
| TPS burst | 580 TPS | **325 TPS** | `013-...md:355, 711` |
| Design limit | 1,000 TPS | 1,000 TPS ✓ | `013-...md:711` |
| Active customers | 30M | **20M** | `013-...md:357` |
| Concurrent campaigns | 5 | **50** | `013-...md:357` |
| Concurrent connections | 5K/10K/25K | **500 / 2,000 / 5,000** | `013-...md:361` |
| Application worker nodes | 4 | **5** | `013-...md:750` |
| Sizing service baseline | 14 services | **16 services** | `013-...md:750` |
| MongoDB raw | 4.01TB → 12.03TB (3x) | **10.28 TB** (trong tổng 14.25 TB) | `013-...md:402` |
| MariaDB raw | 268.21GB → 804.63GB (3x) | **3.97 TB** | `013-...md:402` |
| Kafka msg/day | ~8M | **~39M** | `013-...md:7685, 8129` |

**Hệ quả:** toàn bộ `PERF-001`…`PERF-004`, `PERF-012`, `PERF-013`, `DATA-001`…`DATA-005`, `DATA-010`…`DATA-021`, `K8S-002`, `K8S-006`, `DOC-005`…`DOC-010`, `KAF-002`, `KAF-009` **không thể kết luận PASS/FAIL** vì expected value trong checklist không khớp tài liệu nguồn.

**Recommended Action (P0, phải làm trước mọi việc khác):** re-baseline checklist theo v7, hoặc ra quyết định chính thức xác nhận baseline nào là canonical. Theo quy tắc §1.1 của chính checklist — "Nếu tài liệu tự mâu thuẫn, đánh dấu `DESIGN_CONFLICT`; không tự chọn giá trị có vẻ đúng" — agent **không** tự chọn.

**Confidence: High.**

---

## 2. Baseline và Document Conflicts (§2)

### Canonical service inventory (DOC-002, DOC-004)

Đếm thực tế trong TKTT (`001-...md:1571-1895`): danh sách đánh số **1→17 production services**, nhưng headline cùng dòng ghi *"16 production microservices (2 gateway + 14 business services)"*.

```
1. cms-api-gateway      2. integration-gateway   3. campaign      4. voucher
5. customer             6. product               7. pricing-engine 8. order
9. redemption          10. rule-engine          11. validation   12. distribution
13. segment            14. ingestion            15. stream-processor 16. kafka-relay
17. metadata           | 18. promix-platform (library) | 19. schema (tool)
```

→ 2 gateway + **15** business = **17**, không phải 16 (2+14). **`DOC-004` = DESIGN_CONFLICT.**

### Đối chiếu design ↔ repo ↔ deployed list

| # | TKTT design | Repo trong estate | Có trong deployed list (`003-...md`) | app-config/production | Kết luận |
|---|---|---|---|---|---|
| 1 | cms-api-gateway | ❌ không có | ❌ | ❌ | **FAIL** — không tồn tại trong scope |
| 2 | integration-gateway | ❌ không có | ❌ | ❌ | **FAIL** — không tồn tại trong scope |
| 3 | campaign | `promotion-campaign` | ✅ pp-campaign | ✅ | PASS |
| 4 | voucher | `promotion-coupon` (artifactId `coupon`) | ✅ pp-coupon | ❌ | PARTIAL — tên khác design |
| 5 | customer | `promotion-customer` | ✅ pp-customer | ✅ | PASS |
| 6 | product | `promotion-product` (+ `vds-promotion-product` trùng) | ✅ pp-product | ✅ | PARTIAL — 2 repo cùng artifactId `product` |
| 7 | pricing-engine | `promotion-pricing-engine` (+ `promotion-discount` cũ) | ✅ pp-discount (pricing-engine) | `promotion-discount` | PARTIAL — 2 repo cùng vai trò |
| 8 | order | `promotion-order` | ✅ pp-order | ✅ | PASS |
| 9 | redemption | `promotion-redemption` | ✅ pp-redemption | ✅ | PASS |
| 10 | rule-engine | `promotion-rule-engine` | ❌ | ❌ | PARTIAL |
| 11 | validation | `promotion-validation` | ❌ | ✅ | PARTIAL |
| 12 | distribution | `promotion-distribution` | ✅ pp-distribution | ✅ | PASS |
| 13 | segment | `promotion-segment` | ❌ | ✅ | PARTIAL |
| 14 | ingestion | `promotion-ingestion` | ✅ pp-ingestion | ✅ | PASS |
| 15 | stream-processor | `promotion-stream-processor` | ❌ | ✅ | PARTIAL |
| 16 | kafka-relay | `promotion-kafka-relay` | ✅ pp-kafka-relay | ✅ | PASS |
| 17 | metadata | `promotion-metadata` | ✅ pp-metadata | ❌ | PARTIAL |
| 18 | promix-platform | `promotion-promix-platform` | — | — | PASS (library) |
| 19 | schema | `schema-service` | ❌ | ❌ | PASS (build-time tool) |
| — | *(không có trong design)* | `promotion-validation-engine` | ❌ | ✅ | **DESIGN_CONFLICT** — deploy production nhưng không có trong TKTT |
| — | *(không có trong design)* | `promotion-vtm-bff` | ❌ | ❌ | **DESIGN_CONFLICT** — service thứ 18 không được mô tả |
| — | *(không có trong design)* | `promotion-performance-testing` | ❌ | ✅ | **DESIGN_CONFLICT** — module test được deploy production |

**`DOC-002` Status: DESIGN_CONFLICT.** Ba nguồn (TKTT, deployed list, app-config) cho ba inventory khác nhau: 17 / 12 / 15.

**`DOC-011` Status: FAIL.** Không tìm thấy bất kỳ retention tier 1 năm hay archive 7 năm nào trong code/config. Chi tiết ở §7.

**`DOC-012` Status: FAIL.** Không tồn tại `Document Conflict Register` trong repo. Bảng bên dưới là bản khởi tạo.

### Document Conflict Register (khởi tạo — cần owner xác nhận)

| # | Subject | Value/Claim A | Source A | Value/Claim B | Source B | Decision Needed |
|---|---|---|---|---|---|---|
| 1 | Số production microservices | 16 (2+14) | `001-...md:528, 1573, 905` | 17 (danh sách đánh số 1–17) | `001-...md:1571-1895` | Chốt con số canonical |
| 2 | Service inventory | 17 services | TKTT | 12 services | `003-1-danh-sach-service-database.md` | Deployed list thiếu 5+ service |
| 3 | Service inventory | 12 services | deployed list | 15 services | `app-config/*/production/` | Đồng bộ 3 nguồn |
| 4 | TPS baseline | 11.6 / 116 / 580 | Checklist §9 | 27 / 65 / 325 | `013-...md:711` | Re-baseline checklist |
| 5 | Transactions/ngày | 1M | Checklist DATA-005 | 2.33M | `013-...md:355` | Re-baseline |
| 6 | Active customers | 30M | Checklist DATA-001 | 20M | `013-...md:357` | Re-baseline |
| 7 | Concurrent campaigns | 5 | Checklist DATA-003 | 50 | `013-...md:357` | Re-baseline |
| 8 | Concurrent connections | 5K/10K/25K | Checklist PERF-013 | 500/2K/5K | `013-...md:361` | Re-baseline |
| 9 | App worker nodes | 3 / 4 | Checklist DOC-006 | 5 | `013-...md:750` | Re-baseline |
| 10 | Tổng storage raw | 12.03TB + 804GB | Checklist DATA-010/011 | 14.25TB (Mongo 10.28 + Maria 3.97) | `013-...md:402` | Re-baseline |
| 11 | Kafka msg/ngày | 8M | Checklist KAF-002 | 39M | `013-...md:7685` | Re-baseline |
| 12 | `validation` database | MongoDB | TKTT Phase 2 | MariaDB (Mongo tắt tường minh) | `app-config/promotion-validation/production/application.yml:2,104-105,133-134` | **Chốt DB ownership** |
| 13 | `kafka-relay` offset store | MariaDB | Checklist SVC-KR-003 | MongoDB | `promotion-kafka-relay/.../model/OffsetRecord.java:21` + deployed list | Chốt |
| 14 | `cashback` transaction states | 8 states | TKTT | **9 states** | `promotion-cashback/.../domain/enums/TransactionStatus.java` | Chốt state model |
| 15 | `order` lifecycle | DRAFT→SUBMITTED→VALIDATED→PAYMENT_PENDING→PAID→PROCESSING→SHIPPED→DELIVERED | Checklist SVC-ORD-002 | DRAFT, PLACED, FULFILLED, PARTIALLY_FULFILLED, CANCELLED, REFUNDED, VOIDED | `promotion-order/.../domain/enums/OrderStatus.java` | **Hoàn toàn khác** |
| 16 | `campaign` lifecycle | draft→published→running→stopped | Checklist SVC-CAM-002 | INITIALIZING, ACTIVE, RUNNING, DISABLED, ENABLING, DISABLING, UPDATING, EXPIRED, ERROR, DELETING | `promotion-campaign/.../domain/enums/CampaignStatus.java` | Chốt |
| 17 | Spring Boot baseline | 3.3.6 | `001-...md:565` | 3.5.6 (kafka-relay) | `promotion-kafka-relay/pom.xml` | ADR hoặc align |
| 18 | promix-starter-parent | 1.0.0-SNAPSHOT (18 services) | poms | 2.0.0-SNAPSHOT (coupon) | `promotion-coupon/pom.xml` | Version drift |
| 19 | Service discovery | Eureka | TKTT INT-003 | Eureka (registry của hệ ewallet) | `app-config/*/production:eureka.defaultZone` | Xác nhận shared registry |
| 20 | `validation-engine` | không có trong TKTT | — | deploy production | `app-config/promotion-validation-engine/production/` | Bổ sung vào TKTT |
| 21 | `bff-vtm` | không có trong TKTT | — | repo tồn tại | `promotion-vtm-bff/pom.xml` | Bổ sung hoặc loại |
| 22 | `performance-testing` | module test | — | deploy production | `app-config/promotion-performance-testing/production/` | Xác nhận có chủ đích |
| 23 | 2 gateway services | bắt buộc trong TKTT | `001-...md:358-359` | không tồn tại trong estate | — | **Chốt: đã bỏ hay ở repo khác?** |

---

## 3. Kiến trúc (§3 ARCH)

| ID | Status | Evidence |
|---|---|---|
| ARCH-001 | **PASS** | 23 Maven repo độc lập, mỗi repo 1 `@SpringBootApplication`, DB riêng theo `app-config/*/production` |
| ARCH-002 | **PASS** | Package root riêng `vn.viettel.vds.promotion.<context>` cho từng service |
| ARCH-003 | **PARTIAL** | 17/19 service có layering `adapter / application / domain`. **Ngoại lệ:** `promotion-kafka-relay` dùng `controller/service/repository/model` (`promotion-kafka-relay/src/main/java/vn/viettel/vds/promotion/relay/`) |
| ARCH-004 | **PARTIAL** | Domain sạch ở 13/16 service. **Rò rỉ framework:** `rule-engine` 100/129 file domain import Spring (`@Component` ×87, `@Service` ×13, `@Scheduled`, `@ConfigurationProperties`); `validation-engine` 31/54; `validation` 6/91; `campaign` 4/49 |
| ARCH-005 | **PASS** | `port/in` tồn tại ở 17 service (campaign 70, coupon 56, pricing-engine 56, segment 37, customer 35…) |
| ARCH-006 | **PASS** | `port/out` tồn tại ở 17 service (coupon 20, redemption 18, validation 17, order 15…) |
| ARCH-007 | **FAIL** | Không có `AggregateRoot` / `@Aggregate` ở **bất kỳ** service nào. Domain events chỉ ở `validation` (5 file) và `cashback` (2). Value objects chỉ ở `validation` (4), `coupon` (2). Package tên `domain` **không đủ** làm evidence theo §1.1 |
| ARCH-008 | **PARTIAL** | Campaign/Order/Customer tồn tại như entity + service, không như aggregate có invariant boundary |
| ARCH-009 | **PASS** | 18 service dùng Kafka: `@KafkaListener` 38 file, `KafkaTemplate/StreamBridge` 43 file |
| ARCH-010 | **PASS** | 108 Avro `.avsc` trong `schema-service/src/main/avro/`; **18/18 service** khai báo `<artifactId>schema</artifactId>` trong pom |
| ARCH-011 | **NOT_VERIFIABLE** | Có outbox pattern (xem PLAT-008) nhưng không có event store / replay API. Không có claim event sourcing trong code |
| ARCH-012 | **PARTIAL** | Tách `port/in/command` + `port/in/query` ở `validation`, `metadata`, `order`, `segment`, `product`. Không có write/read model tách storage |
| ARCH-013 | **PARTIAL** | Elasticsearch chỉ là application search ở `customer` (`CustomerSearchDocument`, `CustomerSearchRepository`) và `segment`. **Không phải** CQRS read projection |
| ARCH-014 | **PASS** | Mỗi service 1 DB riêng: `promotion_campaign`, `promotion_cashback`, `promotion_customer`… (`app-config/*/production/application.yml`). Không có cross-service DB write |
| ARCH-015 | **PASS** | Circuit breaker ở 8 service (validation 16 file, redemption 11, order 8); retry ở 15; timeout ở 18; Resilience4j 2.3.0 trong BOM |
| ARCH-016 | **FAIL** | **Không có file OpenAPI/contract nào** trong toàn estate (`find -name "openapi*"` → 0). Springdoc chỉ sinh runtime từ code → code-first, không phải API-first |
| ARCH-017 | **FAIL** | Không có Pact/Spring Cloud Contract trong bất kỳ pom nào |
| ARCH-018 | **NOT_VERIFIABLE** | Không có deployment topology thực tế để so sánh với context diagram |

---

## 4. Service Inventory (§4)

### 4.1 Gateway & Integration — `SVC-GW-001`…`010`, `SVC-IGW-001`…`008`

**Status: FAIL (SVC-GW-001, SVC-IGW-001) / NOT_APPLICABLE (phần còn lại)**

`cms-api-gateway` và `integration-gateway` **không tồn tại** trong estate: không repo, không pom, không Dockerfile, không app-config. Không có Spring Cloud Gateway dependency ở bất kỳ pom nào.

**Gap/Risk (P0):** Toàn bộ control được TKTT đặt ở gateway — JWT validation, OAuth2/OIDC, rate limiting, CORS, IP whitelist, partner authentication, API versioning — hiện **không có nơi nào thực thi trong scope đã audit**. Xem §12 để thấy hệ quả.

### 4.2 Core business services

| ID | Status | Evidence |
|---|---|---|
| SVC-VCH-001 | **PASS** | `promotion-coupon`, 329 java file, lifecycle coupon code + inventory |
| SVC-VCH-002 | **PASS** | `PatternGrammar.java`, `ChunkProcessor.java`, `ChunkGenerateConsumer.java` (bulk qua Kafka chunk) |
| SVC-VCH-003 | **PASS** | `db/changelog/002-create-coupon-codes-table.xml:17` — `<constraints nullable="false" unique="true"/>`; `RedisBloomFilterAdapter.java` chống collision |
| SVC-VCH-004 | **PASS** | `db/changelog/042-create-coupon-code-cleanup-queue-table.xml`, `031-create-processed-messages-table.xml` (dedup theo `consumer_group, message_id`) |
| SVC-VCH-005 | **PASS** | `CampaignServiceAdapter.java`, 6 `@KafkaListener` |
| SVC-VCH-006 | **PASS** | MariaDB — deployed list `003-...md`: `pp-coupon MariaDB coupon` |
| SVC-VCH-007 | **NOT_VERIFIABLE** | Không có performance test report nào cho 100K codes/minute |
| SVC-CAM-001 | **PASS** | `promotion-campaign` 344 file; cashback campaign qua `SagaOrchestrationEnableCashbackConfiguration.java` |
| SVC-CAM-002 | **DESIGN_CONFLICT** | `CampaignStatus.java`: INITIALIZING, ACTIVE, RUNNING, DISABLED, ENABLING, DISABLING, UPDATING, EXPIRED, ERROR, DELETING — **không có** draft/published/stopped |
| SVC-CAM-003 | **PARTIAL** | `CampaignCommon.avsc`: `totalBudget`, `remainingBudget`, `userBudgetLimit` tồn tại — **nhưng kiểu `double`** (xem Critical Finding #4) |
| SVC-CAM-004 | **NOT_VERIFIABLE** | Không xác minh được day-of-week/hour constraint từ scan này |
| SVC-CAM-005 | **PASS** | 5 saga orchestrator: `CampaignSagaCreate/Update/Delete/Enable/DisableOrchestrator.java`; compensation qua `SagaEvents.COMPENSATION_COMPLETE` (`DiscountEventConsumer.java:285-288`); `SagaStatus` = IN_PROGRESS/COMPLETED/ABORTED/ERROR |
| SVC-CAM-006 | **PASS** | Feign clients tới validation/discount/cashback (`app-config/promotion-campaign/production/application.yml:5-15`), 9 `@KafkaListener` |
| SVC-CAM-007 | **PASS** | `jdbc:mariadb:loadbalance://…/promotion_campaign` (`app-config/promotion-campaign/production/application.yml:29`) |
| SVC-CUS-001 | **PASS** | `promotion-customer` 141 file, 35 in-port |
| SVC-CUS-002 | **PASS** | `CustomerSearchDocument.java`, `CustomerSearchRepository.java`, `CustomerSearchPersistenceAdapter.java`; ES config `app-config/promotion-customer/production/application.yml:65-66` |
| SVC-CUS-003 | **PASS** | `batch_size: 500` (`app-config/promotion-customer/production/application.yml:58`) — khớp design |
| SVC-CUS-004 | **NOT_VERIFIABLE** | Không tìm thấy VoucherWarehouse trong customer |
| SVC-CUS-005 | **PARTIAL** | `enable-dlq: true` + retry config (`:194-196`), outbox 23 file. Không xác minh được manual-ack |
| SVC-CUS-006 | **PASS** | `jdbc:mariadb:loadbalance://…/promotion_customer` (`:32`) |
| SVC-PRD-001 | **PASS** | `promotion-product` 251 file, 43 in-port |
| SVC-PRD-002…003 | **NOT_VERIFIABLE** | Không xác minh được MANUAL/AUTOMATIC collection |
| SVC-PRD-004 | **PASS** | Redisson cache config trong `app-config/promotion-product/production/application.yml` |
| SVC-PRD-005 | **PASS** | `jdbc:mariadb:loadbalance://…/promotion_product` (`:26`) |

### 4.3 Financial / transaction services

| ID | Status | Evidence |
|---|---|---|
| SVC-CB-001 | **PASS** | `TransactionStatus.java` — state machine tồn tại |
| SVC-CB-002 | **DESIGN_CONFLICT** | TKTT nói 8 states; enum thực tế **9**: CREATED, PENDING, VALIDATING, REJECTED, AUTHORIZED, PROCESSING, SUCCESS, FAILED, REVERSED |
| SVC-CB-003 | **PASS** | `LedgerAccountEntity.java`, `LedgerEntryEntity.java`, `LedgerEntryType.java` = {CREDIT, DEBIT}, `LedgerEntryState.java`, 2 repository |
| SVC-CB-004 | **PASS** | `BudgetReservationEntity.java`, `BudgetCounterEntity.java`, `BudgetReservationState.java`, `BudgetInitializationService.java` |
| SVC-CB-005 | **PASS** | Outbox 15 file; `retention-period: 7d` (`app-config/promotion-cashback/production/application.yml:272`) |
| SVC-CB-006 | **PARTIAL** | `CashbackTransactionEntity.idempotencyKey` + `findByIdempotencyKey()`. **Nhưng** key đến từ payload upstream, không phải compose `orderId + cashbackId` tại chỗ |
| SVC-CB-007 | **PASS** | MongoDB `promotion_cashback` (`app-config/promotion-cashback/production/application.yml:15-21`) |
| SVC-CB-008 | **PARTIAL** | 4 Mongock changeUnit. **Không có TTL index nào** |
| SVC-DSC-001 | **PASS** | `promotion-pricing-engine` 367 file, 56 in-port |
| SVC-DSC-002 | **PARTIAL** | EvalEx có trong `promotion-pricing-engine/pom.xml:102` và `promotion-discount/pom.xml:198` — **version do BOM quản lý, không pin tường minh**, không xác nhận được 3.1.0 |
| SVC-DSC-003 | **PARTIAL** | Trong code: `BigDecimal` ở 59 file pricing-engine, 41 discount, 41 cashback, 81 redemption ✅. **Nhưng event contract dùng `double`** — xem Critical Finding #4 |
| SVC-DSC-004 | **PASS** | `ValidateDiscountFormulaService` (commit `9babe5d`) |
| SVC-DSC-005 | **NOT_VERIFIABLE** | — |
| SVC-DSC-006 | **PASS** | `jdbc:mariadb:loadbalance://…/promotion_discount` (`app-config/promotion-discount/production/application.yml:29`) |
| SVC-ORD-001 | **PASS** | `OrderStatus.java` tồn tại |
| SVC-ORD-002 | **FAIL** | States thực tế: DRAFT, PLACED, FULFILLED, PARTIALLY_FULFILLED, CANCELLED, REFUNDED, VOIDED. **0/8 state của design khớp** ngoài DRAFT |
| SVC-ORD-003 | **NOT_VERIFIABLE** | — |
| SVC-ORD-004 | **PARTIAL** | Feign + Kafka có; payment integration không thấy trong order |
| SVC-ORD-005 | **PASS** | MongoDB `promotion_order` host `10.101.60.98:8017` (`app-config/promotion-order/production/application.yml:15-18`) |
| SVC-ORD-006 | **PASS** | Outbox 5 file, `retention-period: 7d` (`:244`) |
| SVC-RED-001 | **PARTIAL** | `RedeemStackableDiscountService.java` có `@Transactional`; `SessionRepositoryAdapter`, `RedemptionLogRepositoryAdapter`. **Nhưng ledger nằm ở cashback (MongoDB khác)** → không có atomic boundary chung giữa consume và ledger |
| SVC-RED-002 | **PASS** | `RedemptionController.java` — validate subject type/key |
| SVC-RED-003 | **NOT_VERIFIABLE** | — |
| SVC-RED-004 | **PASS** | `RedeemStackableDiscountService`, `ValidateStackableDiscountResultEvent.avsc` |
| SVC-RED-005 | **PASS** | `idempotencyKey` trong `RedemptionController.java:48`, 34 file có idempotency |
| SVC-RED-006 | **PASS** | `SessionEntity.java:92` — `@Indexed(expireAfterSeconds = 0)`; `ReservationEntity.expiresAt` |
| SVC-RED-007 | **PASS** | MongoDB `promotion_redemption` |
| SVC-RED-008 | **NOT_VERIFIABLE** | — |

### 4.4 Validation services

| ID | Status | Evidence |
|---|---|---|
| SVC-VE-001 | **PASS** | `promotion-validation-engine/pom.xml:82-102` — 5 Drools/KIE artifact |
| SVC-VE-002 | **PASS** | `<drools.version>10.1.0</drools.version>` (`promotion-validation-engine/pom.xml:19`, `promotion-rule-engine/pom.xml:19`) — **khớp chính xác design** |
| SVC-VE-003 | **PASS** | `CompilationCacheService.java`, `KieSessionManager.java`, `DroolsRuleEngineAdapter.java`, `BundlePreloadService.java` |
| SVC-VE-004 | **PASS** | `BundlePreloadService.java`, `CompilationCacheService.java` (TTL cấu hình được) |
| SVC-VE-005 | **PARTIAL** | Prometheus endpoint có; operator discovery không xác minh được |
| SVC-VE-006 | **FAIL** | Production DB = **MariaDB** `promotion_validation_engine` (`app-config/promotion-validation-engine/production/application.yml:25`). TKTT **không mô tả service này**, nên không có expected value để đối chiếu |
| SVC-VAL-001 | **PASS** | `promotion-validation` 438 file |
| SVC-VAL-002 | **PASS** | 108 Avro schema dùng chung qua `schema-service` |
| SVC-VAL-003 | **NOT_VERIFIABLE** | — |
| SVC-VAL-004 | **NOT_VERIFIABLE** | — |
| SVC-VAL-005 | **PASS** | Bundle hash lookup (`7adb1ce` "Fall back to rule-level bundleHash"); `BundleHashResolver.java` ở redemption |
| SVC-VAL-006 | **PARTIAL** | Bundle hash có; immutability không xác minh được |
| SVC-VAL-007…008 | **NOT_VERIFIABLE** | — |
| **SVC-VAL-009** | **FAIL** | **MariaDB, không phải MongoDB.** Config nói tường minh: `# Database Strategy: JPA/MariaDB ONLY (MongoDB disabled)` (`app-config/promotion-validation/production/application.yml:2`), `mongo.enabled: false` (`:104-105`), `# MongoDB Configuration - DISABLED` (`:133-134`) |

### 4.5 Data processing services

| ID | Status | Evidence |
|---|---|---|
| SVC-DIS-001 | **PASS** | 1 `@KafkaListener` + `ProcessDistributionBatchService.java` |
| SVC-DIS-002…003 | **PASS** | `ProviderService.java`, `adapter/out/token/strategy/OAuth2Strategy.java`, `batch/processor/extractor/DataExtractor.java` |
| SVC-DIS-004 | **PASS** | MongoDB `promotion_distribution` (`app-config/promotion-distribution/production/application.yml:11-16`) + MariaDB batch DB (`:103`) — khớp deployed list `distribution + distribution_batch` |
| SVC-DIS-005 | **PASS** | `enable-dlq: true`, 5 Mongock changeUnit |
| SVC-DIS-006 | **NOT_APPLICABLE** | Không có Debezium dependency |
| SVC-SEG-001 | **PASS** | `promotion-segment` 272 file, 37 in-port |
| SVC-SEG-002 | **PASS** | `GenerateCsvFromFile.java`; `max-file-size: 1048576` (`app-config/promotion-segment/production/application.yml:283`) |
| SVC-SEG-003 | **PARTIAL** | `adapter/out/elasticsearch/query` có; historical membership không xác minh |
| SVC-SEG-004 | **PASS** | MongoDB `promotion_segment` (`:32-37`) + MariaDB batch (`:48`) |
| SVC-SEG-005 | **PASS** | 1 `@KafkaListener` + 2 producer |
| SVC-ING-001 | **PASS** | `promotion-ingestion` 89 file |
| SVC-ING-002…003 | **PASS** | 9 Mongock changeUnit; Avro schema validation |
| SVC-ING-004 | **PASS** | `enable-dlq: true`; 13 file liên quan DLQ |
| SVC-ING-005 | **PASS** | MariaDB `promotion_ingestion` (`:15`) |
| SVC-SP-001 | **PASS** | `KafkaStreamsConfig.java` |
| SVC-SP-002 | **PASS** | `RedisDeduplicationStore.java` (Redisson `RBucket`); `deduplication.ttl: 24h` (`app-config/promotion-stream-processor/production/application.yml:299-303`) — **khớp chính xác design** |
| SVC-SP-003 | **PASS** | `enable-dlq: true` |
| SVC-SP-004 | **PASS** | `PiiMasker.java`; `TransactionProcessingService.java:158-171` — `enrichAndMaskTransaction()` → `piiMasker.maskSensitiveData()` trong flow chính |
| SVC-SP-005 | **PARTIAL** | `KafkaStreamsConfig.java:39` — `${promix.kafka.streams.processing.guarantee:exactly_once_v2}` là **default value**; production config **không set tường minh** → phụ thuộc default, không có evidence enforce |
| SVC-SP-006 | **PASS** | 3 `@FeignClient` |
| SVC-SP-007 | **PASS** | MariaDB `promotion_stream_processor` (`:15`) + Redis dedupe |
| SVC-SP-008 | **PARTIAL** | `metrics/` package + prometheus endpoint; không xác minh được metric cụ thể |
| SVC-KR-001 | **PASS** | `promotion-kafka-relay` — source/destination cluster |
| SVC-KR-002 | **NOT_VERIFIABLE** | — |
| SVC-KR-003 | **DESIGN_CONFLICT** | `OffsetRecord.java:21` — `@Document(collection = "offset_records")` → **MongoDB**, không phải MariaDB. Khớp deployed list nhưng khác checklist |
| SVC-KR-004 | **PASS** | `ProcessedMessage.java` + `ProcessedMessageRepository.java` |
| SVC-KR-005 | **PARTIAL** | Retry có (4 file); **`enable-dlq` KHÔNG được set** trong production config |
| SVC-KR-006 | **PASS** | `OffsetRepository.findBy(topic, partition, consumerGroup)` |
| SVC-KR-007 | **PASS** | `RelayMetricsService.java` |

---

## 5. Giao tiếp & Integration (§5 INT)

| ID | Status | Evidence |
|---|---|---|
| INT-001 | **PASS** | `@FeignClient` ở 8 service (validation 9, redemption 7, pricing-engine 5) |
| INT-002 | **PASS** | Kafka ở 18 service |
| INT-003 | **PASS** | Eureka enabled production: `defaultZone: http://production-ewallet-service-registry-api-java.production:8080/eureka/,...-2-...` — **2 registry node**, dùng chung với hệ ewallet |
| INT-004 | **NOT_APPLICABLE** | Không có API Gateway trong scope |
| INT-005 | **NOT_VERIFIABLE** | `cashback.core-transfer` gọi ewallet core gateway (`app-config/promotion-cashback/production/application.yml:332`); không thấy webhook inbound |
| INT-006 | **PASS** | `ProviderService.java`, `OAuth2Strategy.java` (distribution) |
| INT-007 | **NOT_VERIFIABLE** | — |
| INT-008 | **FAIL** | Xem §12 — không service nào enforce JWT/OAuth |
| INT-009 | **PASS** | Timeout config ở 18/18 service (redemption 28 file, validation 33, campaign 38) |
| INT-010 | **PASS** | Idempotency ở 17/18 service (coupon 42, redemption 34, cashback 23) |
| INT-011 | **PARTIAL** | `enable-dlq: true` ở **13/15** production service. **Thiếu: `promotion-kafka-relay`, `promotion-validation-engine`** |
| INT-012 | **PARTIAL** | Avro + `schema-registry` config (campaign, cashback, distribution, customer, discount, perf-testing). **Không tìm thấy compatibility mode (BACKWARD/FORWARD) nào được set** |

---

## 6. Technology stack (§6 TECH)

| ID | Status | Evidence |
|---|---|---|
| TECH-001 | **PASS** | `<java.version>21</java.version>` (`promotion-promix-platform/parent/promix-parent/pom.xml`); kế thừa bởi 19 service. `kafka-relay`, `metadata`, `schema-service` set `21` tường minh |
| TECH-002 | **PARTIAL** | `<spring-boot.version>3.3.6</spring-boot.version>` cho 19 service ✅. **`promotion-kafka-relay/pom.xml` = `spring-boot-starter-parent 3.5.6`** — deviation không có ADR |
| TECH-003 | **PASS** | `<spring-cloud.version>2023.0.3</spring-cloud.version>` + `spring-cloud-kubernetes 2023.0.3` |
| TECH-004 | **PASS** | `promotion-campaign-cms` — Next.js 14.2.29 + React 18.3.1 + TypeScript |
| TECH-005 | **PASS** | IP nội bộ 10.101.60.x cho MariaDB/MongoDB; registry nội bộ `nexus.digital.vn`; không có endpoint cloud công cộng |
| TECH-006 | **NOT_VERIFIABLE** | `spring-cloud-kubernetes` + `promix-discovery-kubernetes-*` module tồn tại, nhưng **production dùng Eureka**, và **không có K8s manifest nào trong estate** |
| TECH-007 | **NOT_VERIFIABLE** | Không tìm thấy config Istio nào |
| TECH-008 | **PASS** | 10/15 production service dùng MariaDB 3-node loadbalance |
| TECH-009 | **PASS** | 7 service dùng MongoDB |
| TECH-010 | **PASS** | Redisson 3.37.0, `mode: CLUSTER`, `default-ttl: 30m` ở 14/15 service |
| TECH-011 | **PASS** | Kafka ở 18 service |
| TECH-012 | **PASS** | ES cho application search ở `customer` + `segment`; ELK riêng cho log (Kibana trong danh sách công cụ `006-1-ci-cd-jobs.md`) |

---

## 7. Data ownership, storage, retention/TTL (§7 DATA)

### 7.1–7.2 Volume & allocation

`DATA-001`…`DATA-021`: **DESIGN_CONFLICT** — xem Critical Finding ở đầu báo cáo. Checklist dùng baseline cũ (1M/day, 30M customers, 12.03TB); tài liệu v7 dùng 2.33M/day, 20M customers, 14.25TB. Không thể PASS/FAIL cho tới khi re-baseline.

### 7.3–7.4 Retention & TTL — **khu vực yếu nhất của hệ thống**

| ID | Status | Evidence |
|---|---|---|
| DATA-030 | **FAIL** | Không tìm thấy retention 1 năm cho financial/compliance data ở bất kỳ đâu |
| DATA-031 | **FAIL** | Không tìm thấy archive 7 năm |
| DATA-032 | **FAIL** | Không tìm thấy retention 6 tháng |
| DATA-033 | **PARTIAL** | Chỉ có outbox retention: `retention-period: 7d` (cashback `:272`, order `:244`, validation `:151`); customer `completed-retention-days: 7` / `failed-retention-days: 30` (`:270-271`) |
| DATA-034 | **PASS** | 7d retention cho processed events (như trên) |
| DATA-035 | **PASS** | Redisson `default-ttl: 30m`, `max-idle-time: 15m`; redemption decision token `ttl: PT5M`, bundle redis TTL `PT30M`, assignment `PT15M` |
| **DATA-036** | **FAIL** | **Chỉ có DUY NHẤT 1 TTL index thật trong toàn hệ thống**: `promotion-redemption/.../entity/SessionEntity.java:92` — `@Indexed(expireAfterSeconds = 0)`, khớp `database/promotion-redemption/promotion-redemption-mongodb-schema.js:97` (`session_ttl_idx`). Tất cả TTL khác là **Caffeine/Redis cache TTL**, không phải data retention |
| DATA-037 | **FAIL** | Không có archive flow (hot→warm→cold), không có object storage integration trong production config |
| DATA-038 | **FAIL** | Không có implementation nén/archive nào |
| DATA-040 | **FAIL** | `order_details` — không có TTL index |
| DATA-041 | **FAIL** | `operational_events` — không có TTL index |
| DATA-042 | **FAIL** | Cashback transaction details — không có TTL index (4 Mongock changeUnit, 0 TTL) |
| DATA-043 | **FAIL** | Wallet snapshots/audit — không có TTL |
| DATA-044 | **FAIL** | Redemption audit decisions — không có TTL |
| DATA-045 | **FAIL** | Validation trails / counter / session (ngoài session chính) — không có TTL |
| DATA-046 | **PARTIAL** | TTL session tồn tại nhưng **driven bởi field `expiresAt`**; giá trị 60 phút không được set trong production config (chỉ thấy `default-ttl: 30m` cho cache) |
| DATA-047 | **PASS** (do thiếu TTL) | Không có TTL ngắn nào đe dọa financial collection — nhưng vì lý do sai: **không có TTL nào cả** |
| DATA-048 | **PARTIAL** | Unique index có ở coupon (Liquibase changelog); Mongock changeUnit tạo index ở cashback/order/segment/distribution/ingestion. Không đủ evidence về compound index phục vụ query pattern chính |
| DATA-049 | **PARTIAL** | PiiMasker ở stream-processor ✅. Không có PII policy/masking ở 17 service còn lại |

**Gap/Risk (P0):** Thiết kế mô tả retention phân tầng 7 ngày → 1 năm (v7) hoặc 30 ngày → 7 năm (checklist). Thực tế chỉ có outbox cleanup 7/30 ngày và 1 session TTL. Với 840M transaction/năm, dữ liệu sẽ tăng vô hạn cho tới khi hết storage — và không có compliance archive.

---

## 8. Kafka / Redis / Elasticsearch / Network (§8)

| ID | Status | Evidence |
|---|---|---|
| KAF-001 | **NOT_VERIFIABLE / RISK** | Không có broker config trong estate. Evidence duy nhất tìm được: `default-replication-factor: 1` trong `app-config/promotion-performance-testing/production/application.yml:331` và `KafkaStreamsConfig.java:48` mặc định `replication-factor:1`. Nếu đây là giá trị áp dụng thật → **FAIL vs RF=3** |
| KAF-002, KAF-003, KAF-009 | **DESIGN_CONFLICT** | 8M vs 39M msg/day; 580 vs 325 TPS burst |
| KAF-004 | **NOT_VERIFIABLE** | `RETENTION_MS_KEY` có trong `promotion-coupon/.../config/KafkaTopicConfig.java:22` nhưng giá trị không xác định được từ config |
| KAF-005 | **PASS** | `acks: all` **và** `enable-idempotence: true` trong production config của: campaign (`:170,176`), cashback (`:154,160`), customer (`:143,149`), discount (`:148,154`), distribution (`:123,129`), ingestion (`:104,110`), order (`:111,117`), perf-testing (`:275`). Evidence mạnh |
| KAF-006 | **NOT_VERIFIABLE** | Partition strategy không có trong config |
| KAF-007 | **PARTIAL** | 13/15 service có `enable-dlq: true`, `dlq-topic-suffix: _dlq`. Thiếu kafka-relay, validation-engine |
| KAF-008 | **PARTIAL** | Avro + schema-registry config có; **compatibility mode không được set** ở bất kỳ đâu |
| REDIS-001 | **PARTIAL** | `redisson.mode: CLUSTER` ở 14 service. Node list đến từ env → không verify được topology thật |
| REDIS-002…003 | **NOT_VERIFIABLE** | — |
| REDIS-004 | **PASS** | `default-ttl: 30m`, `max-idle-time: 15m`; stream-processor dedup `ttl: 24h` |
| REDIS-005 | **NOT_VERIFIABLE** | Không có RDB/AOF config trong estate |
| REDIS-006 | **PASS** | Redis chỉ dùng cache/dedup/session. Financial state ở MongoDB (cashback ledger) / MariaDB |
| ES-001 | **PASS** | `CustomerSearchRepository`, `segment/adapter/out/elasticsearch/query` |
| ES-002 | **NOT_VERIFIABLE** | Không có index template/ILM policy trong repo |
| ES-003 | **PASS** | ES application index (customer/segment) tách khỏi ELK logging (Kibana ở tooling) |
| ES-004 | **NOT_APPLICABLE** | ES không dùng làm CQRS read model |
| NET-001…003 | **NOT_VERIFIABLE** | Không có infra access |
| NET-004 | **PARTIAL** | MariaDB `jdbc:mariadb:loadbalance://` qua 3 node (10.101.60.82/83/84:4006) → có redundant path ở DB tier. **MongoDB chỉ 1 host** |
| NET-005 | **NOT_VERIFIABLE** | Không có firewall/NetworkPolicy trong estate |

---

## 9. Performance & SLO (§9 PERF)

`PERF-001`…`PERF-004`, `PERF-010`…`PERF-014`: **DESIGN_CONFLICT** (baseline cũ vs v7).

| ID | Status | Evidence |
|---|---|---|
| PERF-005 | **NOT_VERIFIABLE** | Target `<100ms` có trong doc (`013-...md:361`); **không có kết quả đo nào** |
| PERF-006 | **NOT_VERIFIABLE** | Target `<500ms` có trong doc; không có kết quả đo |
| PERF-007 | **NOT_VERIFIABLE** | — |
| PERF-008 | **PARTIAL** | Target 99.9% có ở 3 chỗ trong doc (`:265, 364, 736`) nhất quán. **Không có SLO/SLI definition, không có measurement window, không có alerting rule** |
| PERF-009 | **NOT_APPLICABLE** | Không tìm thấy 99.95% trong tài liệu v7 |
| PERF-015…PERF-018 | **FAIL** | **Không tồn tại load test report nào trong toàn estate.** `promotion-performance-testing` là một Spring Boot service (167 java file, 113 avsc) dùng để *sinh tải*, không phải nơi chứa kết quả. Không có `.jtl`, không có Gatling/k6/JMeter report, không có file kết quả trong `05-5-test/` |

**Gap/Risk (P0):** Mọi con số throughput/latency trong TKTT hiện là **design assumption**, chưa có measured evidence. Theo §23 của checklist, không được kết luận đạt chỉ vì có resource limit.

---

## 10. Kubernetes / sizing / autoscaling (§10 K8S)

**`K8S-001` … `K8S-020`: NOT_VERIFIABLE (toàn bộ 20 check).**

Lệnh `grep -rl "kind: Deployment|kind: HorizontalPodAutoscaler|kind: NetworkPolicy|kind: PodDisruptionBudget"` trên toàn estate trả về **0 kết quả**. Không có Helm chart, không có `values.yaml`, không có kustomize.

Nghịch lý cần giải quyết: `006-1-ci-cd-jobs.md` liệt kê **ArgoCD** trong tooling production, nghĩa là manifest tồn tại ở một GitOps repo **ngoài** estate này.

Evidence duy nhất liên quan K8s trong estate:
- `Dockerfile` ở 18/20 service (100% có `HEALTHCHECK`) → hỗ trợ `K8S-014` một phần
- `promix-discovery-kubernetes-spring-boot-starter` tồn tại trong platform nhưng **không service nào dùng** (production dùng Eureka)

**Required action:** cấp quyền đọc GitOps repo + `kubectl` read-only để đóng 20 check này.

---

## 11. Scalability roadmap (§11 SCALE)

| ID | Status | Evidence |
|---|---|---|
| SCALE-001…004 | **PARTIAL** | Doc v7 có roadmap 3x→5x (`013-...md:355, 402`). Không có bottleneck analysis có số liệu |
| SCALE-005 | **PARTIAL** | Sharding key candidate `customer_id` có trong doc (`013-...md:7877`). **Không có đánh giá cardinality/skew**; MongoDB hiện chưa shard (1 host) |
| SCALE-006 | **PARTIAL** | Doc có vertical limit; horizontal ưu tiên — nhưng không verify được |
| SCALE-007 | **FAIL** | Không có owner, không có trigger threshold nào được document |

---

## 12. Security (§12) — **KHU VỰC RỦI RO CAO NHẤT**

### 12.1 API & application security

| ID | Status | Evidence |
|---|---|---|
| **SEC-001** | **FAIL** | **0 HTTPS trong toàn bộ 15 production config.** Tất cả 40 URL nội bộ đều `http://`. MariaDB: `useSSL=false` (campaign `:29`, customer `:32`, discount `:29`, ingestion `:15`, stream-processor `:15`, validation `:15`, validation-engine `:25`). MongoDB/Kafka/Redis: không có TLS param. Không có `server.ssl` (trừ perf-testing). Không có service mesh để bù |
| SEC-002 | **NOT_VERIFIABLE** | Encryption at rest là thuộc tính storage layer |
| SEC-003 | **PARTIAL** | `PiiMasker` chỉ ở stream-processor. 17 service còn lại không có masking |
| SEC-004 | **PARTIAL** | Bean Validation + Avro schema validation có. Không có input sanitization layer tường minh |
| SEC-005 | **NOT_VERIFIABLE** | — |
| SEC-006 | **PARTIAL/FAIL** | Chỉ 3/15 service set limit. campaign: `max-file-size: 10MB` nhưng **`max-request-size: 100MB`** (`:78-79`) — gấp 10× design default 10MB. segment: `max-http-request-header-size: 1048576`, `max-file-size: 1048576`. **12 service không set giới hạn nào** |
| SEC-007 | **PARTIAL** | `characterEncoding=utf8` trong JDBC URL; không có content-type validation |
| SEC-008 | **PARTIAL** | Resilience4j rate limiter chỉ ở cashback (`:532`), redemption (`:498`), perf-testing (`:219`) — đây là **client-side outbound protection**, không phải API ingress rate limiting. Không có gateway |
| SEC-009 | **NOT_APPLICABLE** | Không có authentication endpoint trong scope |
| SEC-010 | **NOT_VERIFIABLE** | Thuộc tầng infra |
| SEC-011 | **FAIL** | Không có HSTS, không có CORS config ở bất kỳ service nào (grep `cors|allowed-origins` → 0 hit) |
| **SEC-012** | **FAIL** | `springdoc` + `swagger-ui.path` được cấu hình trong **cả 15** production config, **không có `enabled: false` ở đâu**, `promotion-product` còn set `enabled: true` tường minh (`:66`). API docs phơi ra ở production |

### 12.2 Identity & authorization

| ID | Status | Evidence |
|---|---|---|
| **SEC-ID-001** | **FAIL** | **Không repo nào có Spring Security.** `grep -rln "spring-boot-starter-security\|spring-security\|oauth2-resource-server" */pom.xml` → **0 kết quả**. Không có `SecurityConfig`/`WebSecurityConfigurer` nào tồn tại. Không có `issuer-uri`/`jwk-set-uri` trong 15 production config. Các hit "JWT" chỉ là: (a) `README.md` mẫu trong `campaign/adapter/config/` và `cashback/adapter/config/`, (b) `redemption/.../DecisionTokenService.java` — token nghiệp vụ nội bộ cho fast-path, **không phải** user authentication |
| SEC-ID-002 | **FAIL** | Không có refresh/revocation/logout |
| SEC-ID-003 | **FAIL** | Không có OAuth2/OIDC IdP config. `cashback.core-transfer.auth` là **client credentials đi ra** ewallet core, không phải inbound authN |
| SEC-ID-004 | **NOT_APPLICABLE** | — |
| SEC-ID-005 | **NOT_APPLICABLE** | — |
| SEC-ID-006 | **FAIL** | `@PreAuthorize/hasRole/hasAuthority` xuất hiện ở 9 file nhưng **không có Spring Security trên classpath → annotation không được enforce**. Đây chính xác là trường hợp §23 cảnh báo |
| SEC-ID-007 | **NOT_APPLICABLE** | Không có OPA integration (chỉ 1 hit ngẫu nhiên ở segment) |
| **SEC-ID-008** | **FAIL** | **`tenantId` đến từ REQUEST BODY, không từ token.** `RedemptionController.java:48,61,70,74,81,93` — `request.tenantId()`. Không có filter/interceptor nào validate tenant claim. Caller tự khai tenant nào cũng được |
| SEC-ID-009 | **NOT_VERIFIABLE** | Không có mTLS config; không có service mesh |
| **SEC-ID-010** | **FAIL** | 124 secret dùng `${ENV_VAR}` ✅ **nhưng 2 secret hardcode trong production config đã commit vào git**: `app-config/promotion-redemption/production/application.yml:262` → `redemption.decision-token.secret` (literal 39 ký tự) và `app-config/promotion-cashback/production/application.yml:336` → `cashback.core-transfer.auth.client-secret` (literal, credential gọi **ewallet core transfer / chuyển tiền**). Không có Vault integration ở đâu (`grep -rli vault app-config/promotion-*` → 0) |

> **Lưu ý bảo mật:** giá trị secret không được in ra trong báo cáo này theo §1.1. Cần rotate cả hai và purge khỏi git history.

### 12.3 Zero Trust

| ID | Status | Evidence |
|---|---|---|
| SEC-ZT-001 | **FAIL** | Không có authN/authZ ở boundary của bất kỳ service nào |
| SEC-ZT-002 | **PARTIAL** | Mỗi service có DB user riêng (`authentication-database: promotion_<svc>`) ✅. Kafka ACL / service account không verify được |
| SEC-ZT-003 | **NOT_VERIFIABLE** | Không có NetworkPolicy trong estate |
| SEC-ZT-004 | **NOT_VERIFIABLE** | Không có Istio config |
| SEC-ZT-005 | **NOT_VERIFIABLE** | — |

### 12.4 Security testing

| ID | Status | Evidence |
|---|---|---|
| SEC-TST-001 | **FAIL** | **0/20 Jenkinsfile có SAST/SCA/DAST.** `grep -ciE 'sonar\|trivy\|snyk\|dependency-check\|owasp\|clair\|grype'` trên tất cả Jenkinsfile → 0 |
| SEC-TST-002 | **FAIL** | Không có image scanning stage nào |
| SEC-TST-003…007 | **NOT_VERIFIABLE** | Thuộc quy trình tổ chức, ngoài repo |

---

## 13. Multi-tenancy (§13 TEN)

| ID | Status | Evidence |
|---|---|---|
| **TEN-001** | **FAIL** | Không có tenant strategy nào được implement. **`promix-tenant-core` chỉ chứa 4 file — toàn bộ là exception class**: `TenantContextMissingException`, `TenantDataSourceException`, `TenantIsolationViolationException`, `TenantNotFoundException`. Không có `TenantContext`, không có resolver, không có data-source routing. `promix-tenant-autoconfigure` chỉ có `TenantExceptionAdviceAutoConfiguration` + `TenantExceptionHandler` |
| **TEN-002** | **FAIL** | `tenantId` chỉ là một field payload được truyền tay qua method signature (redemption 902 lần, validation/validation-engine tương tự). Không có context propagation qua HTTP filter, Kafka header, hay background job. `TenantContext` duy nhất tìm thấy là **inner class Drools fact**: `validation-engine/.../execution/FactPreparationService.java:351` |
| TEN-003 | **FAIL** | Không có cross-tenant test nào (10/19 service có 0 test file) |
| **TEN-004** | **FAIL** | Không có authorization layer → không thể enforce tenant permission |
| TEN-005 | **NOT_APPLICABLE** | Shared DB, không có per-tenant pool |
| TEN-006 | **FAIL** | Không có per-tenant quota |
| TEN-007 | **FAIL** | Không có tenant config isolation |
| **TEN-008** | **FAIL** | 13/18 service **không có tenant dimension nào**: campaign, coupon, customer, product, pricing-engine, order, distribution, segment, ingestion, stream-processor, metadata, cashback, kafka-relay. Cache key Redisson và Kafka key không mang tenant → nguy cơ cross-tenant leak |

---

## 14. Backup / HA / DR (§14 DR)

| ID | Status | Evidence |
|---|---|---|
| **DR-001** | **FAIL (client-side)** | **Cả 7 service MongoDB đều trỏ vào MỘT host duy nhất `10.101.60.98:8017`** (cashback `:17`, distribution `:12`, kafka-relay `:17`, order `:17`, redemption, segment `:33`, perf-testing). Dùng `host:`/`port:` chứ không phải `uri:` với `replicaSet=`. **Không có `replicaSet`, `readPreference`, hay `w=majority` ở bất kỳ config nào.** Dù server có là replica set, client cũng không failover được |
| DR-002 | **PASS** | MariaDB 3-node: `jdbc:mariadb:loadbalance://10.101.60.82:4006,10.101.60.83:4006,10.101.60.84:4006` ở 10 service |
| DR-003 | **NOT_VERIFIABLE / RISK** | Xem `KAF-001` — evidence duy nhất là RF=1 |
| DR-004 | **PARTIAL** | `redisson.mode: CLUSTER` + `retry-attempts: 3`, `retry-interval: 1500`. Topology không verify được, chưa test |
| DR-005…DR-008 | **NOT_VERIFIABLE** | Không có backup job/script/cron nào trong estate |
| DR-009 | **NOT_VERIFIABLE** | Không có PITR evidence |
| DR-010 | **NOT_VERIFIABLE** | RPO 5 phút có trong doc (`013-...md:736`); không có test |
| DR-011 | **NOT_VERIFIABLE** | RTO 15 phút có trong doc; không có test |
| DR-012 | **NOT_VERIFIABLE** | — |
| DR-013 | **NOT_VERIFIABLE** | "<30 giây" có trong doc (`013-...md:402`); mâu thuẫn với DR-001 ở phía client MongoDB |
| DR-014 | **NOT_VERIFIABLE** | — |
| DR-015…DR-017 | **NOT_VERIFIABLE** | Không có backup/restore test evidence |
| DR-018 | **PARTIAL** | `@Transactional` ở redemption/cashback; ledger CREDIT/DEBIT. **Nhưng MongoDB không set `w=majority`** → durability của financial write không được đảm bảo ở mức config |

---

## 15. Observability (§15 OBS)

| ID | Status | Evidence |
|---|---|---|
| OBS-001 | **PASS** | `management.endpoints.web.exposure.include: health,info,metrics,prometheus` ở **15/15** production service |
| OBS-002 | **NOT_VERIFIABLE** | Grafana có trong tooling list; không có dashboard JSON trong repo |
| OBS-003 | **NOT_VERIFIABLE** | Không có AlertManager config/alert rule nào trong estate |
| OBS-004 | **PASS** | `CorrelationIdInterceptor.java` (redemption); tracing `propagation.type: b3` + `sampling.probability: 1.0` ở tất cả service |
| OBS-005 | **NOT_VERIFIABLE** | ELK/Kibana ngoài repo |
| **OBS-006** | **FAIL** | **Distributed tracing bị tắt export ở production.** 14/15 service: `zipkin.tracing.enabled: false  # Completely disable Zipkin` + `endpoint: ""` + `spring.autoconfigure.exclude: ZipkinAutoConfiguration`. Chỉ `promotion-validation-engine` set `true` (và chỉ để "required for Tracer bean"). → trace ID có trong log nhưng **không span nào tới backend** |
| OBS-007 | **NOT_APPLICABLE** | Không có Sentry |
| OBS-008 | **NOT_VERIFIABLE** | — |
| OBS-009 | **PARTIAL** | Micrometer expose Kafka consumer metric, nhưng không có alert rule cho lag |
| OBS-010 | **PARTIAL** | HikariCP metric qua actuator; không có alert |
| OBS-011 | **PASS** | JVM/GC metric mặc định qua `prometheus` endpoint |
| OBS-012 | **PASS** | `http.server.requests` histogram mặc định. cashback/redemption còn expose `circuitbreakers,ratelimiters,retries,bulkheads` |
| OBS-020…OBS-027 | **NOT_VERIFIABLE** | **Không có alert threshold nào được định nghĩa ở đâu trong estate** (8 check) |

---

## 16. CI/CD (§16 CICD)

Tất cả 20 Jenkinsfile có cùng một shape. Mẫu `promotion-redemption/Jenkinsfile`:

```groovy
sh "mvn clean package -DskipTests"          // ← bỏ qua toàn bộ test
sh "docker build -t ${DOCKER_IMAGE} ."
docker login … && docker push               // → harleyhoang/<service>  (Docker Hub cá nhân)
sh "docker stop ${SERVICE} || true; docker rm ${SERVICE} || true;
    docker run -d --name ${SERVICE} --restart=always --network promotion-network …"
```

| ID | Status | Evidence |
|---|---|---|
| CICD-001 | **PASS** | 30 git repo; branch gate `env.GIT_BRANCH == 'origin/main'` |
| CICD-002 | **PASS** | `mvn clean package` |
| **CICD-003** | **FAIL** | **`-DskipTests` ở 18/20 Jenkinsfile.** Unit test không bao giờ chạy trong pipeline |
| **CICD-004** | **FAIL** | JaCoCo chỉ có ở **2/19** service (`promotion-metadata`, `promotion-kafka-relay`). **10/19 service có 0 test file**: campaign, customer, order, redemption, validation, ingestion, stream-processor, cashback, kafka-relay, vtm-bff. Không có coverage gate |
| CICD-005 | **FAIL** | Không có integration test stage |
| CICD-006 | **FAIL** | Không có security scanning (0/20) |
| CICD-007 | **PARTIAL** | Image tag `v1.${GIT_COMMIT[0..6]}` ✅ traceable. Nhưng build không reproducible (không pin base image digest) |
| CICD-008 | **FAIL** | Không có image scan |
| CICD-009 | **PARTIAL** | Nexus có trong tooling doc và `promotion-ui-core/Jenkinsfile` (`NEXUS_URL`). **Java service push lên Docker Hub cá nhân `harleyhoang/*`** |
| **CICD-010** | **FAIL** | Không dùng Harbor. Registry là `harleyhoang/<service>` trên Docker Hub — **image production trên tài khoản cá nhân public registry** |
| CICD-011 | **FAIL** | Không có Helm/IaC. Deploy bằng `docker run` thủ công trong Jenkinsfile |
| CICD-012 | **FAIL** | Không có Vault. Credential qua Jenkins credential store + env var; 2 secret hardcode (xem SEC-ID-010) |
| CICD-013 | **PARTIAL** | Liquibase (campaign, coupon, customer, discount, metadata, pricing-engine, product, rule-engine, stream-processor, validation, validation-engine) + Mongock (cashback 4, distribution 5, ingestion 9, order 4, redemption 2, segment 7) ✅. Nhưng chạy lúc app startup, không phải stage riêng có kiểm soát |
| CICD-014 | **PARTIAL** | `HEALTHCHECK` trong 18/20 Dockerfile ✅. Không có smoke test stage |
| CICD-015 | **FAIL** | Không có rollback stage. `docker rm` trước `docker run` → **không rollback được, và có downtime** |
| CICD-016 | **FAIL** | Không có canary/blue-green |

> **Ghi chú quan trọng:** `006-1-ci-cd-jobs.md` liệt kê ArgoCD, Sonar, Nexus, Harbor-like tooling cho môi trường Product. Rất có thể **pipeline production thật nằm ở Jenkins/GitLab ngoài estate này**, và Jenkinsfile in-repo chỉ là dev pipeline. **Cần xác nhận** — nếu đúng, các check trên chuyển thành `NOT_VERIFIABLE` thay vì `FAIL`, nhưng Jenkinsfile dev đang push image lên Docker Hub cá nhân thì vẫn là rủi ro cần xử lý.
>
> **Thêm:** `006-1-ci-cd-jobs.md` (trang PDF 3887) chứa **credential dạng plaintext** (tài khoản bitbucket và password Nexus của jenkins) ngay trong tài liệu bàn giao. Cần rotate + xóa khỏi tài liệu.

---

## 17. Promix-platform (§17 PLAT)

| ID | Status | Evidence |
|---|---|---|
| PLAT-001 | **PASS** | Aggregator pom: *"Reactor build cho toàn bộ thư viện nội bộ. Aggregator này không phát hành."* Không có Dockerfile/Jenkinsfile → không deploy như service |
| PLAT-002 | **PASS** | 70 module đúng cấu trúc: `bom/` (1) → `parent/` (6) → `core/` (20) → `adapters/` (2) → `autoconfigure/` (21) → `starters/` (19) → `starters-test/` (1) |
| PLAT-003 | **PASS** | `uuid-creator.version 6.1.1` trong BOM (UUID v7 support) |
| PLAT-004 | **PARTIAL** | `promix-security-core`, `promix-security-autoconfigure`, `promix-security-webmvc-adapter`, `promix-security-webflux-adapter`, `promix-security-spring-boot-starter` tồn tại đầy đủ — **nhưng 0 service nào dùng** |
| PLAT-005 | **FAIL** | Không có OPA end-to-end flow |
| PLAT-006 | **PASS** | `promix-validation-core` + `promix-error-core` + `promix-error-autoconfigure`; `promix.error.mode: hybrid` dùng ở production |
| PLAT-007 | **PASS** | `promix-cache-core`, `promix-reactive-cache-core` + autoconfigure + starter; Redisson 3.37.0; dùng thực tế ở 14 service |
| PLAT-008 | **PASS** | `promix-messaging-core` + `promix-outbox-core` + `promix-outbox-jpa-core` + autoconfigure + starter. Outbox dùng ở 13 service (validation 26, coupon 23, customer 23, product 16, cashback 15) |
| **PLAT-009** | **FAIL** | `promix-tenant-core` **chỉ có 4 exception class**, không có tenant context/data-source routing. Xem §13 |
| PLAT-010 | **PASS** | `promix-resilience-core` + autoconfigure + starter; Resilience4j 2.3.0; circuit breaker/retry/ratelimiter/bulkhead expose qua actuator ở cashback |
| PLAT-011 | **PASS** | `promix-web-core`, `promix-web-mvc-adapter`, `promix-web-flux-adapter`, `promix-error-core` |
| PLAT-012 | **PARTIAL** | `promix-observer-core` + autoconfigure + starter; Micrometer tracing 1.3.6, Brave 6.0.3 ✅. **Nhưng export bị tắt** (OBS-006) |
| PLAT-013 | **PARTIAL** | `promix-bom` quản lý 34 version tập trung ✅. **Drift: `promotion-coupon` dùng parent + schema `2.0.0-SNAPSHOT`** trong khi 18 service khác dùng `1.0.0-SNAPSHOT`; `kafka-relay` bỏ hẳn promix parent |
| PLAT-014 | **PARTIAL** | 21 autoconfigure module với `@ConditionalOnProperty`. Test coverage của platform không đo được (không có JaCoCo) |
| PLAT-015 | **PARTIAL** | `BUILD.md`, `README.md`, `docs/` tồn tại. Override không được document tập trung |

---

## 18. Schema governance (§18 SCHEMA)

| ID | Status | Evidence |
|---|---|---|
| SCHEMA-001 | **PASS** | `schema-service/pom.xml` — `spring-boot-starter-parent 3.3.6` trực tiếp, không có Dockerfile/Jenkinsfile. Artifact `schema:1.0.0-SNAPSHOT` |
| SCHEMA-002 | **PASS** | **108 file `.avsc`** trong `schema-service/src/main/avro/` (campaign, cashback, common, customer, discount, engine, ingestion, order, product, redemption, schema) — đều trong version control |
| SCHEMA-003 | **PARTIAL** | Nullable-with-default pattern được dùng đúng (`"type":["null",…],"default":null` — backward-compatible). **Nhưng không có compatibility mode nào được set ở schema registry config** |
| SCHEMA-004 | **PASS** | Generated class trong `schema-service/src/main/generated/`; avro-maven-plugin; Avro 1.11.3 + Confluent 7.5.1 trong BOM |
| **SCHEMA-005** | **PASS** | **18/18 service** khai báo `<artifactId>schema</artifactId>`; 17 dùng `1.0.0-SNAPSHOT`. ⚠️ `promotion-coupon` dùng `2.0.0-SNAPSHOT` → **producer/consumer có thể lệch version** |

> ⚠️ **Duplication risk:** `promotion-performance-testing` chứa **113 file `.avsc` riêng** (bản copy của schema-service). Hai nguồn schema song song → nguy cơ drift.

---

## 19. Data governance / audit / compliance (§19 GOV)

| ID | Status | Evidence |
|---|---|---|
| GOV-001 | **FAIL** | Không có data classification (Public/Internal/Confidential/Restricted) ở đâu |
| GOV-002 | **FAIL** | PII/payment field không được map classification |
| GOV-003 | **PARTIAL** | DB user tách theo service ✅; encryption không có (SEC-001/002) |
| GOV-004 | **PARTIAL** | `created_at/updated_at` trong entity; không có lineage |
| GOV-005 | **PARTIAL** | Bean Validation + Avro schema + unique constraint. Không có data-quality rule framework |
| GOV-006 | **PASS** | Mỗi master entity thuộc đúng 1 service+DB (xem ARCH-014) |
| GOV-007 | **FAIL** | Không có right-to-access/export (`grep -rliE "gdpr\|erasure\|consent"` → **0 hit toàn estate**) |
| GOV-008 | **FAIL** | Không có erasure workflow |
| GOV-009 | **FAIL** | Không có consent tracking |
| **GOV-010** | **FAIL** | **Không có audit log implementation nào** (`grep -rlni "auditlog\|audit_log\|AuditEntry\|@Audit"` → 0 hit). Chỉ có `campaign/domain/enums/audit/SagaStatus.java` (saga state, không phải audit log) |
| GOV-011 | **FAIL** | Không có tamper-resistant/write-once control |
| GOV-012 | **FAIL** | Không có archive 7 năm (trùng DATA-031) |
| GOV-013 | **PARTIAL** | Cashback ledger (CREDIT/DEBIT + LedgerEntryState) là audit trail tài chính tốt nhất hiện có. **Nhưng không có retention bảo vệ, không có tamper-resistance, không có search interface** |
| GOV-014 | **FAIL** | Không có compliance report/export |
| GOV-015 | **FAIL** | Không có mapping GDPR/PCI DSS/ISO 27001 nào tới control thực tế |

---

## Critical Findings (xếp theo mức độ)

| # | ID | Pri | Status | Finding | Evidence | Risk | Action |
|---|---|---|---|---|---|---|---|
| 1 | SEC-ID-001/006/008, INT-008, SEC-ZT-001 | P0 | FAIL | **Không service nào có authentication/authorization.** 0 repo có Spring Security; 0 `SecurityConfig`; 0 `issuer-uri`. `@PreAuthorize` ở 9 file **không được enforce** vì không có security trên classpath | `grep -rln "spring-boot-starter-security\|oauth2-resource-server" */pom.xml` → 0 | Mọi API nghiệp vụ và tài chính (redemption, cashback, order) không xác thực | Triển khai gateway hoặc bật `promix-security-spring-boot-starter` (đã có sẵn, chưa dùng) |
| 2 | SEC-ID-010, CICD-012 | P0 | FAIL | **2 secret hardcode trong production config đã commit git**, gồm credential gọi **ewallet core transfer** | `app-config/promotion-cashback/production/application.yml:336`; `app-config/promotion-redemption/production/application.yml:262` | Lộ credential chuyển tiền | **Rotate ngay**, chuyển sang env/Vault, purge git history |
| 3 | TEN-001/002/004/008, PLAT-009, SEC-ID-008 | P0 | FAIL | **Multi-tenancy không được implement.** `promix-tenant-core` chỉ có 4 exception class. `tenantId` đến từ **request body** chứ không từ token | `promix-tenant-core/.../exception/` (4 file, hết); `RedemptionController.java:48,70` | Caller tự khai tenant → cross-tenant data access | Implement tenant resolver từ token claim + filter + enforce ở repository |
| 4 | SVC-DSC-003, SVC-CAM-003 | P0 | FAIL | **Tiền tệ dùng `double` trong event contract.** `Money.avsc` làm đúng (string→BigDecimal) nhưng 4 schema khác dùng `double`: `totalBudget`, `remainingBudget`, `userBudgetLimit`, `maxCashbackAmount`, `cashbackPercentage`, `discountValue`, `maxDiscountAmount`, `minOrderAmount`, `order.totalAmount`, `effectiveDiscountRate` | `CampaignCommon.avsc`, `CampaignCreatedEvent.avsc`, `OrderCreatedEventFlat.avsc`, `ValidateStackableDiscountResultEvent.avsc` | Sai số tích lũy trên budget và discount qua Kafka | Chuyển sang `Money` record hoặc `bytes+decimal` logical type |
| 5 | DR-001, DR-013, DR-018 | P0 | FAIL | **Cả 7 service MongoDB trỏ 1 host duy nhất**, không `replicaSet=`, không `w=majority`, không `readPreference` | `host: 10.101.60.98` / `port: 8017` ở 7 production config | Mất node = mất toàn bộ order/cashback/redemption/segment; không đảm bảo durability financial write | Chuyển sang `uri:` với replica set + `w=majority` cho financial DB |
| 6 | DATA-036, DATA-030/031/040-047 | P0 | FAIL | **Chỉ 1 TTL index thật trong toàn hệ thống.** Không có retention 1 năm, không có archive 7 năm, không có archive flow | `SessionEntity.java:92` là TTL index duy nhất | Với 840M tx/năm, storage tăng vô hạn; không đạt compliance | Thiết kế và triển khai retention/archive theo tier |
| 7 | SEC-001 | P0 | FAIL | **Không có TLS ở đâu trong production.** 40/40 URL nội bộ là `http://`; MariaDB `useSSL=false`; không TLS cho Mongo/Kafka/Redis; không service mesh | 15 production config | Credential và dữ liệu tài chính truyền plaintext | Bật TLS hoặc triển khai mTLS qua service mesh |
| 8 | GOV-010/011/012/013 | P0 | FAIL | **Không có audit log.** Không có Who/What/When/Where/Why/How, không tamper-resistance, không archive | `grep "auditlog\|audit_log\|AuditEntry\|@Audit"` → 0 hit | Không truy vết được giao dịch tài chính; không đáp ứng compliance | Triển khai audit log append-only cho financial flow |
| 9 | DOC-000 và 22 conflict khác | P0 | DESIGN_CONFLICT | **Ba inventory khác nhau (17/12/15) và hai baseline sizing khác nhau** | Xem Document Conflict Register | Không có baseline để nghiệm thu | Chốt canonical inventory + baseline bằng ADR |
| 10 | CICD-003/004/006/008/015 | P0 | FAIL | **Pipeline không chạy test, không scan, không rollback.** `-DskipTests` ở 18/20; 10/19 service có 0 test file; image lên Docker Hub cá nhân `harleyhoang/*` | `*/Jenkinsfile` | Thay đổi vào production không có bất kỳ gate chất lượng/bảo mật nào | Xác nhận pipeline production thật; nếu đúng pipeline này → khắc phục toàn bộ |
| 11 | SVC-VAL-009 | P0 | FAIL | `validation` dùng **MariaDB**, TKTT yêu cầu **MongoDB** — và config nói tường minh "MongoDB disabled" | `app-config/promotion-validation/production/application.yml:2,104-105,133-134` | Sizing MongoDB sai; TKTT sai | Cập nhật TKTT hoặc migrate |
| 12 | SVC-ORD-002 | P0 | FAIL | Order lifecycle **hoàn toàn khác** design (7 state khác vs 8 state thiết kế) | `OrderStatus.java` | Tài liệu không dùng được để nghiệm thu | Cập nhật TKTT theo implementation |
| 13 | PERF-015…018 | P0 | FAIL | **Không có load test report nào tồn tại** | Toàn estate: 0 `.jtl`/Gatling/k6 report | Mọi con số TPS/latency là assumption | Chạy load test theo baseline v7 (27/65/325 TPS) |
| 14 | SEC-TST-001/002 | P0 | FAIL | **Không có SAST/SCA/DAST/image scan** trong CI | 0/20 Jenkinsfile | CVE vào production không được phát hiện | Thêm Sonar + Trivy + dependency-check |
| 15 | OBS-006 | P1 | FAIL | **Distributed tracing bị tắt export** ở 14/15 service | `zipkin.tracing.enabled: false` + `endpoint: ""` + autoconfigure exclude | Không debug được hot path 5 service (redemption→validation→rule-engine→pricing-engine→coupon) | Bật lại exporter tới Zipkin (đã có trong tooling) |
| 16 | SEC-012 | P1 | FAIL | **Swagger UI phơi ra ở production** trên cả 15 service | 15 production config; `promotion-product:66` `enabled: true` | Lộ bề mặt API | Tắt springdoc ở production |
| 17 | ARCH-007 | P1 | FAIL | **DDD chỉ có ở tên package.** 0 aggregate root, domain event chỉ ở 2/19 service | `grep "AggregateRoot\|@Aggregate"` → 0 | Invariant nghiệp vụ không có nơi cưỡng chế | Hoặc implement aggregate, hoặc bỏ claim DDD khỏi TKTT |
| 18 | ARCH-016/017 | P1 | FAIL | **Không có file OpenAPI nào**; không có contract test | `find -name "openapi*"` → 0 | "API-First" trong TKTT không đúng thực tế | Sinh và version hóa OpenAPI spec |
| 19 | K8S-001…020 | P1 | NOT_VERIFIABLE | **Không có K8s manifest nào trong estate** dù TKTT yêu cầu K8s và ArgoCD có trong tooling | 0 hit `kind: Deployment` | 20 check P0/P1 không đánh giá được | Cấp quyền GitOps repo |
| 20 | INT-011, KAF-007 | P1 | PARTIAL | `kafka-relay` và `validation-engine` **không có DLQ** trong production | `enable-dlq` vắng ở 2/15 config | Message lỗi mất im lặng ở relay | Bật DLQ |
| 21 | TECH-002, PLAT-013, SCHEMA-005 | P1 | PARTIAL | Version drift: `kafka-relay` Spring Boot 3.5.6; `coupon` promix+schema `2.0.0-SNAPSHOT` | poms | Schema lệch giữa producer/consumer | Align versions |
| 22 | SEC-006 | P1 | PARTIAL | `max-request-size: 100MB` ở campaign — gấp 10× design 10MB; 12 service không giới hạn | `app-config/promotion-campaign/production/application.yml:79` | DoS qua payload lớn | Set global body limit |
| 23 | — | P1 | RISK | `promotion-performance-testing` (module test, 113 avsc trùng lặp) **được deploy production** | `app-config/promotion-performance-testing/production/` | Công cụ sinh tải chạy ở production | Xác nhận có chủ đích hoặc gỡ |
| 24 | — | P2 | RISK | **Credential plaintext trong tài liệu bàn giao** | `006-1-ci-cd-jobs.md` (PDF trang 3887) | Lộ tài khoản bitbucket + Nexus | Rotate + xóa khỏi tài liệu |

---

## Performance & Capacity

| Mục | Kết quả |
|---|---|
| Baseline load (v7) | 27 TPS avg / 65 TPS peak / 325 TPS burst / 1,000 TPS design limit |
| Baseline load (checklist) | 11.6 / 116 / 580 TPS — **stale, xem DOC-000** |
| **Measured load** | **KHÔNG CÓ** |
| p95 / p99 latency | **KHÔNG ĐO** (target `<100ms` / `<500ms` chỉ là design) |
| Error rate | **KHÔNG ĐO** |
| Kafka lag | **KHÔNG ĐO** (không có alert rule) |
| CPU/Mem/IO/Network | **KHÔNG ĐO** (không có K8s resource spec trong estate) |

---

## Security & Compliance

| Mục | Kết quả |
|---|---|
| **AuthN/AuthZ** | **KHÔNG CÓ** ở mọi service |
| **Tenant isolation** | **KHÔNG CÓ**; `tenantId` từ request body |
| **Secret management** | 124 env ref ✅ / **2 secret hardcode trong production** ❌ / không Vault |
| **SAST/SCA/DAST/Image scan** | **KHÔNG CÓ** (0/20 pipeline) |
| **Audit/retention** | **KHÔNG CÓ** audit log; retention chỉ 7–30 ngày cho outbox |
| **TLS** | **KHÔNG CÓ** (100% plaintext nội bộ) |
| **PII masking** | 1/18 service (stream-processor) |

---

## HA / Backup / DR

| Mục | Kết quả |
|---|---|
| MariaDB | 3-node loadbalance ✅ (`10.101.60.82/83/84:4006`) |
| **MongoDB** | **1 host** `10.101.60.98:8017`, không replicaSet, không `w=majority` ❌ |
| Redis | `mode: CLUSTER` + retry ✅ (topology chưa verify) |
| Kafka | Evidence duy nhất: RF=1 ⚠️ |
| **Failover evidence** | **KHÔNG CÓ** |
| **Restore test evidence** | **KHÔNG CÓ** |
| RPO | Target 5 phút (doc), **chưa test** |
| RTO | Target 15 phút (doc), **chưa test** |

---

## Final Verdict

# `INSUFFICIENT_EVIDENCE`

**Lý do (theo §20 Acceptance Gate):**

- **`FINAL-001` FAIL** — chưa có canonical service inventory được ADR/owner xác nhận. Ba nguồn cho ba con số: 17 / 12 / 15.
- **`FINAL-002` FAIL** — còn **23 `DESIGN_CONFLICT` chưa giải quyết**, trong đó có service count, DB ownership (`validation`), security model, retention, và HA/DR critical path.
- **`FINAL-003` FAIL** — nhiều check `P0` ở trạng thái `FAIL` (security, multi-tenancy, retention, audit, MongoDB HA), chưa có risk acceptance chính thức nào.
- **`FINAL-006` FAIL** — không có bất kỳ load/performance evidence nào.
- **`FINAL-007` FAIL** — backup/restore/DR chỉ có target trong tài liệu, không có test evidence.
- **`FINAL-008` FAIL** — security control **không có cả ở mức dependency**, chứ chưa nói runtime.
- **`FINAL-009` PARTIAL** — migration/index evidence có (Liquibase + Mongock), nhưng TTL evidence gần như không tồn tại.

Theo §23: không được kết luận "hệ thống đúng theo TKTT" khi security library tồn tại nhưng endpoint không enforce, khi backup job chưa từng restore thử, và khi tài liệu có nhiều giá trị khác nhau mà chưa chốt.

---

## Required Actions (thứ tự ưu tiên)

### Ngay lập tức (trong tuần)
1. **Rotate 2 secret** tại `app-config/promotion-cashback/production/application.yml:336` và `app-config/promotion-redemption/production/application.yml:262`; purge khỏi git history. Rotate cả credential trong `006-1-ci-cd-jobs.md`.
2. **Xác nhận pipeline production thật** — nếu Jenkinsfile in-repo đang được dùng thật, dừng việc push image production lên Docker Hub cá nhân `harleyhoang/*`.
3. **Xác nhận có gateway/ingress nào đang enforce authentication không.** Nếu không → đây là P0 security incident, không phải gap thiết kế.

### Trước khi nghiệm thu (P0)
4. Ra **ADR chốt canonical service inventory** (17 hay 16 hay 15) và **baseline sizing** (v7: 27/65/325 TPS, 2.33M tx/day, 20M customers). Cập nhật checklist theo baseline đã chốt.
5. **Triển khai authN/authZ**: `promix-security-spring-boot-starter` đã tồn tại sẵn trong platform và chưa service nào dùng — đây là đường ngắn nhất.
6. **Sửa tenant isolation**: resolve `tenantId` từ token claim, không từ request body; thêm filter + enforce ở repository layer.
7. **Sửa MongoDB connection**: dùng `uri:` với replica set, `w=majority` cho cashback/order/redemption.
8. **Đổi kiểu tiền tệ trong Avro** từ `double` sang `Money`/decimal logical type ở 4 schema.
9. **Thiết kế + triển khai retention/TTL/archive** theo tier đã cam kết.
10. **Triển khai audit log** append-only cho financial flow (redemption, cashback, order).
11. **Bật TLS** (hoặc mTLS qua service mesh) cho traffic nội bộ và DB connection.
12. **Chạy load test** theo baseline v7, report đủ: error rate, p95/p99, throughput, resource usage, Kafka lag — gắn với build/config/resource profile cụ thể (`PERF-015`…`018`).
13. **Chạy DR drill**: MongoDB failover, MariaDB failover, restore test. Đo RPO/RTO thật.
14. **Thêm SAST/SCA/image scan** vào pipeline; bật test (`-DskipTests` → chạy test); thêm coverage gate.

### Đóng evidence gap (P1)
15. **Cấp quyền đọc GitOps repo** (ArgoCD manifest) → đóng 20 check §10 K8s + `SEC-ZT-003` + `NET-004`.
16. **Cấp quyền Prometheus/Grafana/AlertManager** → đóng `OBS-002/003/005/008/020-027` (11 check).
17. **Cấp quyền DB** → xác minh index/TTL/collection thật (`DATA-040`…`048`).
18. Bật DLQ cho `kafka-relay` và `validation-engine`.
19. Bật lại tracing exporter (`OBS-006`); tắt Swagger ở production (`SEC-012`).
20. Align version: `kafka-relay` Spring Boot 3.5.6 → 3.3.6; `coupon` promix/schema 2.0.0 → 1.0.0 (hoặc nâng tất cả).
21. Sinh và version hóa OpenAPI spec (`ARCH-016`); thêm contract test (`ARCH-017`).
22. Set schema-registry compatibility mode tường minh (`KAF-008`, `SCHEMA-003`).
23. Quyết định về `promotion-discount` vs `promotion-pricing-engine`, và `promotion-product` vs `vds-promotion-product` (mỗi cặp là 2 repo cùng vai trò).
24. Bổ sung `validation-engine`, `bff-vtm` vào TKTT hoặc loại khỏi production.

---

## Appendix — Evidence Access Gaps (`FINAL-005`)

Danh sách quyền truy cập còn thiếu để đóng 168 check `NOT_VERIFIABLE`:

| Cần | Đóng được check |
|---|---|
| GitOps repo (ArgoCD manifests) | `K8S-001`…`020`, `SEC-ZT-003/004`, `TECH-006/007`, `NET-004/005` (~26) |
| `kubectl` read-only production | `K8S-*` runtime, `DR-001`…`004` (~12) |
| Prometheus + Grafana + AlertManager | `OBS-002/003/005/008/009/010/020-027`, `PERF-005`…`014` (~25) |
| DB session (MongoDB + MariaDB) | `DATA-036`…`048`, `SVC-*-00x` DB checks (~20) |
| Kafka admin (broker config, topic describe) | `KAF-001`…`004`, `006`, `008` (~6) |
| Backup system + DR runbook | `DR-005`…`017` (~13) |
| Jenkins/GitLab CI production jobs | `CICD-*` xác nhận lại (~16) |
| Load test environment + report | `PERF-001`…`018` (~18) |
| Security scanning tooling (Sonar, Trivy) | `SEC-TST-001`…`007` (~7) |

---

*Báo cáo tuân theo §1.1 của checklist: không dùng tên file/class/comment làm evidence duy nhất; không in giá trị secret; phân biệt design capacity với measured capacity; phân biệt code capability với feature enabled in production.*
