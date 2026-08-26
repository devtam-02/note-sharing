# LOAD TESTING REPORT - SAMPLE TEMPLATE

## 1. Executive Summary

**Mục tiêu:** Xác nhận hệ thống có đáp ứng tải mục tiêu và SLO hay không, đồng thời xác định bottleneck và capacity hiện tại.

| Hạng mục | Kết quả |
|---|---|
| Target Load | 2,000 TPS |
| Sustainable Capacity | ~1,900 TPS |
| P95 tại target | 480 ms |
| P99 tại target | 900 ms |
| Error Rate | 0.10% |
| Bottleneck chính | Database |
| Kết luận | Đạt sát target, chưa đủ headroom cho production |

**Graph cần có:** Không bắt buộc, nhưng nên có 1 graph tổng hợp TPS + P95/P99 theo thời gian nếu báo cáo dùng để trình bày.

---

## 2. Test Objective & Acceptance Criteria

### 2.1. Mục tiêu test

- Xác nhận hệ thống xử lý được **2,000 TPS**.
- Xác nhận latency và error rate vẫn nằm trong SLO.
- Xác định saturation point.
- Xác định bottleneck chính.
- Đánh giá khả năng recovery sau peak.

### 2.2. Acceptance Criteria

| Metric | Target |
|---|---:|
| Throughput | >= 2,000 TPS |
| P95 | < 500 ms |
| P99 | < 1,000 ms |
| Error Rate | < 0.1% |
| App CPU | < 75% |
| DB CPU | < 85% |
| Kafka Lag | Không tăng liên tục |
| Memory | Ổn định, không leak |
| Restart / OOM | 0 |

**Graph cần có:** Không cần. Bảng là đủ.

---

## 3. Test Environment

| Component | Configuration |
|---|---|
| Application | 4 pods |
| CPU / Pod | 2 vCPU limit |
| Memory / Pod | 3 GiB limit |
| JVM | Java 21 |
| Database | MariaDB, 8 vCPU, 32 GB RAM |
| DB Pool | 30 connections / pod |
| Redis | 3 nodes |
| Kafka | 6 partitions |
| Consumer | 6 instances |
| Load Tool | k6 / JMeter / Gatling |

**Graph cần có:** Không cần.

---

## 4. Workload Model

### 4.1. Tổng tải

| Thông số | Giá trị |
|---|---:|
| Total Target TPS | 2,000 |
| Test Duration | 60 phút |
| Ramp-up | 10 phút |
| Steady State | 40 phút |
| Recovery | 10 phút |

### 4.2. Traffic Mix

| API / Flow | Tỷ lệ | Target TPS |
|---|---:|---:|
| GET Campaign | 40% | 800 |
| Validate | 20% | 400 |
| Cashback | 15% | 300 |
| Create Campaign | 10% | 200 |
| Search | 10% | 200 |
| Other | 5% | 100 |

Công thức:

```text
Endpoint TPS = Total TPS × Traffic Ratio
```

**Graph cần có:** Không bắt buộc. Nếu traffic mix phức tạp, có thể dùng pie/bar chart.

---

## 5. Test Scenario

| Phase | Duration | Load |
|---|---:|---:|
| Warm-up | 5 phút | 300 TPS |
| Step 1 | 10 phút | 500 TPS |
| Step 2 | 10 phút | 1,000 TPS |
| Step 3 | 10 phút | 1,500 TPS |
| Step 4 | 10 phút | 2,000 TPS |
| Stress | 5 phút | 2,500 TPS |
| Recovery | 10 phút | 1,000 TPS |

**Graph cần có:** **Nên có.**  
Graph TPS theo thời gian giúp chứng minh workload thực tế khớp với kịch bản.

---

## 6. Test Results

### 6.1. API Performance

| Offered TPS | Actual TPS | P50 | P95 | P99 | Error Rate |
|---:|---:|---:|---:|---:|---:|
| 500 | 500 | 80 ms | 120 ms | 180 ms | 0.00% |
| 1,000 | 995 | 90 ms | 160 ms | 250 ms | 0.00% |
| 1,500 | 1,480 | 110 ms | 230 ms | 400 ms | 0.02% |
| 2,000 | 1,930 | 180 ms | 480 ms | 900 ms | 0.10% |
| 2,500 | 2,050 | 500 ms | 1.5 s | 4.0 s | 2.00% |

**Nhận xét:**

- Hệ thống đáp ứng gần đủ target 2,000 TPS.
- P95 và P99 tăng mạnh sau khoảng 2,000 TPS.
- Actual TPS gần như không tăng tương ứng khi offered load tăng lên 2,500 TPS.
- Error Rate tăng rõ rệt tại mức stress.

**Graph cần có:** **Bắt buộc nên có.**

Khuyến nghị:
- TPS theo thời gian.
- P95/P99 theo thời gian hoặc theo load step.
- Error Rate theo thời gian.

---

## 7. Application Resource

| Load | App CPU | Memory | GC Pause P99 | Thread Queue |
|---:|---:|---:|---:|---:|
| 500 TPS | 20% | 45% | 20 ms | 0 |
| 1,000 TPS | 35% | 48% | 25 ms | 0 |
| 1,500 TPS | 50% | 52% | 30 ms | 0 |
| 2,000 TPS | 62% | 55% | 35 ms | 5 |
| 2,500 TPS | 65% | 57% | 40 ms | 20 |

**Nhận xét:**

- CPU chưa saturation.
- Memory ổn định.
- GC chưa phải bottleneck chính.
- Thread queue bắt đầu tăng ở mức stress.

**Graph cần có:** **Nên có.**

Ưu tiên:
- CPU theo thời gian.
- Memory/Heap theo thời gian.
- GC Pause nếu nghi ngờ JVM bottleneck.

---

## 8. Database Metrics

| Load | DB CPU | Query P95 | DB Pool Usage | Pending Connections |
|---:|---:|---:|---:|---:|
| 500 TPS | 25% | 35 ms | 30% | 0 |
| 1,000 TPS | 45% | 50 ms | 45% | 0 |
| 1,500 TPS | 65% | 90 ms | 60% | 0 |
| 2,000 TPS | 85% | 180 ms | 85% | 10 |
| 2,500 TPS | 98% | 380 ms | 100% | 120 |

**Nhận xét:**

- DB CPU tăng mạnh theo tải.
- Query latency tăng rõ từ 1,500 TPS.
- Connection pool saturation tại 2,500 TPS.
- Database là bottleneck chính.

**Graph cần có:** **Bắt buộc nên có** nếu kết luận DB là bottleneck.

Ưu tiên:
- DB CPU theo thời gian.
- Query latency.
- Active/Pending DB connections.

---

## 9. Kafka Metrics

| Load | Producer Rate | Consumer Rate | Max Lag | Recovery Time |
|---:|---:|---:|---:|---:|
| 1,000 TPS | 3,000 msg/s | 3,000 msg/s | 0 | 0 |
| 1,500 TPS | 4,500 msg/s | 4,450 msg/s | 500 | < 1 phút |
| 2,000 TPS | 6,000 msg/s | 5,900 msg/s | 5,000 | ~2 phút |
| 2,500 TPS | 7,500 msg/s | 6,200 msg/s | 120,000 | ~15 phút |

Công thức:

```text
Lag Growth Rate = Producer Rate - Consumer Rate
```

```text
Recovery Time
=
Current Lag / (Consumer Capacity - Producer Rate)
```

**Nhận xét:**

- Kafka ổn tới khoảng 2,000 TPS.
- Lag tăng nhanh tại mức stress.
- Consumer/downstream chưa đủ recovery capacity ở tải cao.

**Graph cần có:** **Nên có**, đặc biệt nếu hệ thống event-driven.

Ưu tiên:
- Consumer Lag theo thời gian.
- Producer Rate vs Consumer Rate.

---

## 10. Cache / Redis Metrics

| Metric | Kết quả |
|---|---:|
| Cache Hit Ratio | 95.5% |
| Redis P95 | 4 ms |
| Redis CPU Peak | 55% |
| Eviction | 0 |
| Hot Key | Không phát hiện |
| Big Key | Không phát hiện |

**Nhận xét:**

Redis chưa phải bottleneck.

**Graph cần có:** Không bắt buộc nếu metric ổn định.  
Chỉ cần graph khi hit ratio, latency hoặc memory có biến động bất thường.

---

## 11. Saturation & Bottleneck Analysis

### 11.1. Saturation Point

Saturation bắt đầu khoảng:

```text
~2,000 TPS
```

Bằng chứng:

- Offered TPS tăng nhưng Actual TPS tăng rất ít.
- P95/P99 tăng mạnh.
- Error Rate tăng.
- DB CPU tiến gần 100%.
- DB connection pool xuất hiện pending requests.

### 11.2. Primary Bottleneck

```text
Database
```

### 11.3. Evidence

| Evidence | Kết quả |
|---|---|
| App CPU | ~65%, chưa saturation |
| DB CPU | 98% |
| DB Query P95 | tăng 35 ms -> 380 ms |
| DB Pool | đạt 100% |
| Pending DB Connections | tăng lên 120 |
| API P95 | tăng 480 ms -> 1.5 s |

**Graph cần có:** **Bắt buộc nên có** để chứng minh bottleneck.

---

## 12. Capacity Assessment

### Sustainable Capacity

```text
~1,900 - 2,000 TPS
```

Điều kiện:

```text
P95 < 500ms
P99 < 1s
Error <= 0.1%
```

### Headroom

Nếu production peak:

```text
1,500 TPS
```

và sustainable capacity:

```text
2,000 TPS
```

Công thức:

```text
Headroom
=
(Capacity - Peak)
/
Peak
× 100%
```

```text
=
(2,000 - 1,500)
/
1,500
× 100%
≈ 33.3%
```

| Metric | Value |
|---|---:|
| Production Peak | 1,500 TPS |
| Sustainable Capacity | 2,000 TPS |
| Headroom | ~33% |

**Graph cần có:** Không bắt buộc.

---

## 13. Optimization Recommendations

| Priority | Vấn đề | Hướng xử lý |
|---:|---|---|
| P1 | DB CPU cao | Review slow queries + EXPLAIN |
| P1 | Query latency tăng | Kiểm tra index/composite index |
| P1 | DB pool saturation | Giảm query/transaction duration trước khi tăng pool |
| P2 | Read-heavy traffic | Xem xét cache |
| P2 | Kafka lag khi stress | Tối ưu consumer/downstream |
| P3 | Thread queue tăng | Review blocking calls |

Không nên chỉ:

```text
Scale application pods
```

vì app CPU chưa phải bottleneck.

---

## 14. Re-test Plan

Sau tối ưu:

| Test | Mục tiêu |
|---|---|
| Load Test | Validate lại 2,000 TPS |
| Capacity Test | Xác nhận capacity mới |
| Stress Test | Tìm breaking point mới |
| Soak Test | Kiểm tra stability dài hạn |

Phải giữ:

```text
Same workload
Same data
Same environment
Same duration
```

để so sánh Before/After.

**Graph cần có:** Nên có graph Before vs After cho:
- P95/P99.
- TPS.
- DB CPU.
- Query latency.

---

# 15. Final Conclusion

Ví dụ:

```text
Hệ thống đáp ứng gần đầy đủ target 2,000 TPS nhưng chỉ còn ít margin
trước khi database bắt đầu saturation.

Tại 2,500 TPS:
- Throughput không tăng tương ứng.
- P95/P99 tăng mạnh.
- Error Rate tăng lên 2%.
- DB CPU đạt 98%.
- DB connection pool saturation.

Database được xác định là bottleneck chính.

Khuyến nghị ưu tiên tối ưu query/index/transaction trước khi scale application.
Sau tối ưu cần thực hiện lại Load Test + Stress Test để xác định capacity mới.
```

---

# 16. Checklist Graph nên có trong báo cáo

| Graph | Mức độ cần thiết | Mục đích |
|---|---|---|
| TPS theo thời gian | Bắt buộc nên có | Chứng minh workload và actual throughput |
| P95/P99 theo thời gian | Bắt buộc nên có | Chứng minh latency/SLO |
| Error Rate | Nên có | Chứng minh failure tại peak |
| App CPU | Nên có | Xác định app saturation |
| Heap/Memory | Tùy trường hợp | Chứng minh leak/stability |
| GC Pause | Tùy trường hợp | Chứng minh JVM bottleneck |
| DB CPU | Bắt buộc nếu DB bottleneck | Chứng minh DB saturation |
| DB Pool Active/Pending | Bắt buộc nếu pool bottleneck | Chứng minh queue/wait |
| Kafka Lag | Bắt buộc với async flow | Chứng minh backlog |
| Producer vs Consumer Rate | Nên có | Chứng minh nguyên nhân lag |
| Redis Hit Ratio | Tùy trường hợp | Chứng minh cache effectiveness |
| Before vs After | Nên có khi re-test | Chứng minh hiệu quả tối ưu |

---

# 17. Nguyên tắc trình bày báo cáo

Một báo cáo tốt nên:

```text
Ít chữ
Nhiều bảng
Graph có mục đích
Kết luận có evidence
Không liệt kê metric không liên quan
```

Mỗi kết luận nên đi theo format:

```text
Observation
-> Evidence
-> Conclusion
-> Recommendation
```

Ví dụ:

```text
Observation:
P95 tăng mạnh sau 2,000 TPS.

Evidence:
DB CPU tăng lên 98%, DB pending tăng 0 -> 120,
trong khi App CPU chỉ ~65%.

Conclusion:
Database là primary bottleneck.

Recommendation:
Tối ưu query/index/transaction trước khi scale app.
```
