# SYSTEM SIZING REPORT - SAMPLE TEMPLATE

## 1. Executive Summary

**Mục tiêu:** Xác định capacity và resource cần thiết để hệ thống đáp ứng tải production dự kiến trong giai đoạn mục tiêu.

| Hạng mục | Kết quả |
|---|---:|
| Current Peak TPS | 1,000 TPS |
| Expected Growth | 50% |
| Safety Margin | 30% |
| Target Capacity | ~1,950 TPS |
| Recommended Capacity | ~2,000 TPS |
| App Pods đề xuất | 10-11 pods |
| DB QPS dự kiến | ~10,000 QPS |
| Kafka Throughput | ~6,000 msg/s |
| Bottleneck dự kiến | Database |
| Kết luận | Đủ cho target nếu DB được tối ưu và validate bằng load test |

**Graph cần có:** Không bắt buộc. Nếu trình bày cho management/architecture review, nên có 1 graph thể hiện **Current Peak → Forecast → Target Capacity**.

---

## 2. Business Input

| Thông số | Giá trị |
|---|---:|
| Transactions / Day | 10,000,000 |
| Current Peak TPS | 1,000 |
| Expected Growth | 50% |
| Peak Factor | 8x Average |
| Safety Margin | 30% |
| Read / Write Ratio | 70 / 30 |
| Average Payload | 100 KB |
| Data Retention | 90 ngày |
| Target P95 | < 500 ms |
| Availability Target | 99.9% |

**Graph cần có:** Không cần.

---

## 3. Traffic Capacity Calculation

### 3.1. Average TPS

Công thức:

```text
Average TPS
=
Transactions per Day / 86,400
```

```text
10,000,000 / 86,400
≈ 116 TPS
```

### 3.2. Target Capacity

Dùng peak thực tế:

```text
Current Peak = 1,000 TPS
```

Công thức:

```text
Target Capacity
=
Peak TPS
× Growth Factor
× Safety Margin Factor
```

```text
=
1,000
× 1.5
× 1.3
=
1,950 TPS
```

Làm tròn:

```text
Recommended Target = 2,000 TPS
```

| Metric | Value |
|---|---:|
| Average TPS | ~116 |
| Current Peak TPS | 1,000 |
| Growth Factor | 1.5 |
| Safety Margin Factor | 1.3 |
| Calculated Capacity | 1,950 |
| Recommended Target | 2,000 TPS |

**Graph cần có:** **Nên có** nếu cần giải thích forecast/capacity planning.

---

## 4. Workload Distribution

| Flow | Traffic Ratio | Target TPS |
|---|---:|---:|
| Read Campaign | 40% | 800 |
| Validate | 20% | 400 |
| Cashback | 15% | 300 |
| Create Campaign | 10% | 200 |
| Search | 10% | 200 |
| Other | 5% | 100 |
| **Total** | **100%** | **2,000** |

Công thức:

```text
Flow TPS
=
Total TPS × Traffic Ratio
```

**Graph cần có:** Không bắt buộc. Có thể dùng bar chart nếu nhiều flow.

---

## 5. Dependency Load Estimation

Giả sử trung bình một transaction tạo:

```text
5 DB queries
1 Redis operation
2 HTTP downstream calls
3 Kafka messages
```

| Dependency | Công thức | Estimated Load |
|---|---|---:|
| Database | 2,000 × 5 | 10,000 QPS |
| Redis | 2,000 × 1 | 2,000 OPS |
| HTTP Downstream | 2,000 × 2 | 4,000 RPS |
| Kafka | 2,000 × 3 | 6,000 msg/s |

**Lưu ý:** Các số này là estimate sơ bộ; cần validate bằng tracing/production metrics.

**Graph cần có:** Không cần.

---

## 6. Concurrency Estimation

Giả sử:

```text
Target TPS = 2,000
Average Response Time = 200ms = 0.2s
```

Công thức Little's Law:

```text
Concurrency
≈
TPS × Response Time
```

```text
2,000 × 0.2
=
400 concurrent requests
```

Nếu latency tăng lên:

```text
500ms
```

thì:

```text
2,000 × 0.5
=
1,000 concurrent requests
```

| Scenario | TPS | Avg Latency | Approx Concurrency |
|---|---:|---:|---:|
| Normal | 2,000 | 200 ms | 400 |
| Degraded | 2,000 | 500 ms | 1,000 |

**Graph cần có:** Không cần.

---

## 7. Application Pod Sizing

Benchmark giả định:

```text
1 pod sustainable capacity = 250 TPS
```

Công thức:

```text
Minimum Pods
=
Target TPS / TPS per Pod
```

```text
2,000 / 250
=
8 pods
```

Nếu thêm 30% capacity headroom:

```text
8 × 1.3
=
10.4
```

Đề xuất:

```text
10-11 pods
```

| Metric | Value |
|---|---:|
| Sustainable TPS / Pod | 250 |
| Target TPS | 2,000 |
| Minimum Pods | 8 |
| Headroom | 30% |
| Recommended Pods | 10-11 |

**Graph cần có:** **Nên có** nếu cần trình bày scale curve 2/4/8/10 pods.

---

## 8. CPU Sizing

Giả sử benchmark:

```text
1 pod
2 vCPU
250 TPS
CPU ~60%
```

Đề xuất:

| Resource | Value |
|---|---:|
| CPU Request / Pod | 1 vCPU |
| CPU Limit / Pod | 2 vCPU |
| Expected Peak CPU | ~60-70% |
| Recommended Pod Count | 10-11 |

Không nên sizing để normal peak chạy ở 95-100% CPU.

**Graph cần có:** Không bắt buộc trong sizing report. Nên có nếu sizing dựa trên benchmark thực tế.

---

## 9. Memory / JVM Sizing

Giả sử:

```text
Observed peak RSS/pod = 2.1 GiB
Expected concurrency growth = +30%
```

Đề xuất:

| Resource | Value |
|---|---:|
| Memory Request / Pod | 2 GiB |
| Memory Limit / Pod | 3 GiB |
| JVM Heap Max | ~2.0-2.2 GiB |
| Native/Non-Heap Headroom | ~0.8-1.0 GiB |

Lưu ý:

```text
Container memory != JVM Heap
```

Cần chừa:

- Metaspace
- Thread Stack
- Direct Buffer
- Code Cache
- Native Memory

**Graph cần có:** Không bắt buộc. Nếu có soak-test evidence thì nên đính kèm Heap/RSS graph.

---

## 10. Thread Pool Sizing

Giả sử:

```text
CPU cores = 2
Compute Time = 20ms
Wait Time = 80ms
```

Approximation:

```text
Threads
≈
CPU Cores × (1 + Wait / Compute)
```

```text
=
2 × (1 + 80/20)
=
10 threads
```

Đây chỉ là điểm bắt đầu.

| Metric | Value |
|---|---:|
| CPU Cores / Pod | 2 |
| Compute Time | 20 ms |
| Wait Time | 80 ms |
| Initial Thread Estimate | ~10 |
| Final Value | Validate bằng load test |

**Graph cần có:** Không cần.

---

## 11. Database Capacity Sizing

### 11.1. DB QPS

```text
Target TPS = 2,000
Queries / Transaction = 5
```

```text
DB QPS
=
2,000 × 5
=
10,000 QPS
```

### 11.2. Connection Pool

Giả sử:

```text
Safe DB connection budget = 300
Recommended app pods = 11
```

```text
Pool / Pod
<=
300 / 11
≈ 27
```

Đề xuất:

```text
Hikari maxPoolSize ≈ 20-25/pod
```

để còn headroom.

| Metric | Value |
|---|---:|
| Estimated DB QPS | 10,000 |
| DB Connection Budget | 300 |
| App Pods | 11 |
| Theoretical Pool / Pod | ~27 |
| Recommended Pool / Pod | 20-25 |

### 11.3. Risk

Database có khả năng là shared bottleneck đầu tiên nếu:

- query chưa tối ưu,
- N+1,
- index chưa phù hợp,
- transaction dài,
- lock contention cao.

**Graph cần có:** **Nên có** nếu sizing dựa trên benchmark hiện tại: DB CPU/QPS/Query P95.

---

## 12. Database Storage Sizing

Giả sử:

```text
Rows/day = 5,000,000
Avg Row Size = 500 bytes
Index + Overhead = 80%
Retention = 90 days
```

Raw data/day:

```text
5M × 500 bytes
≈ 2.5 GB/day
```

Including index/overhead:

```text
2.5 × 1.8
≈ 4.5 GB/day
```

Retention:

```text
4.5 × 90
≈ 405 GB
```

Đề xuất thêm headroom:

```text
~500-600 GB usable storage
```

chưa tính backup/replica/binlog.

| Metric | Value |
|---|---:|
| Raw Data / Day | ~2.5 GB |
| Data + Index / Day | ~4.5 GB |
| 90-Day Data | ~405 GB |
| Recommended Usable | 500-600 GB |

**Graph cần có:** Không bắt buộc. Với forecast dài hạn có thể dùng storage growth graph.

---

## 13. Redis Sizing

Giả sử:

```text
Active Keys = 10M
Avg Key + Value = 1 KB
```

Raw:

```text
~10 GB
```

Thêm:

- object overhead,
- fragmentation,
- replication,
- headroom.

Đề xuất:

```text
~20-25 GB usable memory
```

tùy topology.

| Metric | Value |
|---|---:|
| Active Keys | 10M |
| Avg Key+Value | 1 KB |
| Raw Size | ~10 GB |
| Recommended Memory | 20-25 GB |
| Target Hit Ratio | >=95% |

**Graph cần có:** Không cần trong sizing report; dùng nếu muốn chứng minh hit ratio/memory trend.

---

## 14. Kafka Consumer Sizing

Giả sử:

```text
Producer Rate = 6,000 msg/s
1 Consumer Capacity = 1,500 msg/s
```

Công thức:

```text
Consumers Required
=
Producer Rate / Consumer Capacity
```

```text
6,000 / 1,500
=
4 consumers
```

Để 4 consumers active:

```text
Partitions >= 4
```

Khuyến nghị có headroom:

```text
6 partitions
```

| Metric | Value |
|---|---:|
| Producer Rate | 6,000 msg/s |
| Consumer Capacity | 1,500 msg/s |
| Minimum Active Consumers | 4 |
| Minimum Partitions | 4 |
| Recommended Partitions | 6 |

**Graph cần có:** Không bắt buộc. Nếu sizing dựa trên current system thì nên có Producer vs Consumer Rate và Lag graph.

---

## 15. Kafka Storage Sizing

Giả sử:

```text
6,000 msg/s
Avg Message Size = 2 KB
Retention = 3 days
Replication Factor = 3
```

Ingress:

```text
6,000 × 2 KB
=
12 MB/s
```

Per day:

```text
12 MB/s × 86,400
≈ 1.04 TB/day
```

3 days:

```text
~3.1 TB
```

Replication x3:

```text
~9.3 TB
```

Đề xuất thêm operational headroom:

```text
~11-12 TB cluster usable/raw requirement
```

tùy compression và broker layout.

| Metric | Value |
|---|---:|
| Kafka Rate | 6,000 msg/s |
| Avg Message | 2 KB |
| Raw / Day | ~1.04 TB |
| Retention | 3 days |
| Replication | 3 |
| Estimated Storage | ~9.3 TB |
| Recommended | ~11-12 TB |

**Graph cần có:** Không cần.

---

## 16. Network Sizing

Giả sử:

```text
Target RPS = 2,000
Average Response = 100 KB
```

Outbound:

```text
2,000 × 100 KB
=
200 MB/s
```

Nếu có internal fan-out:

```text
2 downstream calls/request
```

network nội bộ còn cao hơn.

| Metric | Value |
|---|---:|
| External RPS | 2,000 |
| Avg Response Size | 100 KB |
| External Egress | ~200 MB/s |
| Internal Calls | ~4,000 RPS |
| Recommendation | Validate NIC/LB/service mesh capacity |

**Graph cần có:** Không bắt buộc.

---

## 17. Queue / Backlog Sizing

Giả sử spike:

```text
Producer = 10,000 msg/s
Consumer = 6,000 msg/s
Duration = 60s
```

Công thức:

```text
Backlog
=
(Input Rate - Processing Rate)
× Duration
```

```text
=
(10,000 - 6,000) × 60
=
240,000 messages
```

Queue cần chịu ít nhất:

```text
240k + headroom
```

| Metric | Value |
|---|---:|
| Burst Input | 10,000 msg/s |
| Processing | 6,000 msg/s |
| Burst Duration | 60 s |
| Backlog Created | 240,000 |
| Recommended Buffer | >300,000 |

**Graph cần có:** Không cần.

---

## 18. Headroom Assessment

Nếu:

```text
Target Production Peak = 1,500 TPS
Sustainable Capacity = 2,000 TPS
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

**Graph cần có:** Không cần.

---

## 19. N-1 / Failover Capacity

Giả sử:

```text
3 zones
Total Capacity = 3,000 TPS
```

Mỗi zone:

```text
~1,000 TPS
```

Nếu mất 1 zone:

```text
Remaining Capacity
=
2,000 TPS
```

Production peak:

```text
1,500 TPS
```

=> N-1 vẫn đủ.

| Scenario | Available Capacity | Peak Load | Status |
|---|---:|---:|---|
| Normal | 3,000 TPS | 1,500 | PASS |
| Lose 1 AZ | 2,000 TPS | 1,500 | PASS |
| Lose 2 AZ | 1,000 TPS | 1,500 | FAIL |

**Graph cần có:** Không cần.

---

## 20. Capacity Forecast

| Period | Peak TPS | Required Capacity | Recommended Pods |
|---|---:|---:|---:|
| Current | 1,000 | 1,300 | 6 |
| 3 Months | 1,300 | 1,690 | 8 |
| 6 Months | 1,500 | 1,950 | 10-11 |
| 12 Months | 2,500 | 3,250 | 15-17 |

**Graph cần có:** **Nên có** cho architecture/capacity review. Line chart Peak TPS vs Required Capacity là đủ.

---

## 21. Final Sizing Summary

| Layer | Proposed Sizing |
|---|---|
| Target Capacity | 2,000 TPS |
| Application | 10-11 pods |
| CPU / Pod | 1 vCPU request / 2 vCPU limit |
| Memory / Pod | 2 GiB request / 3 GiB limit |
| DB QPS | ~10,000 |
| DB Pool | 20-25 / pod |
| DB Storage | ~500-600 GB usable |
| Redis | ~20-25 GB usable |
| Kafka Consumers | 4 minimum |
| Kafka Partitions | 6 recommended |
| Kafka Storage | ~11-12 TB estimated |
| Network Egress | ~200 MB/s external |
| Headroom | ~30-33% |
| N-1 Capacity | PASS |

---

## 22. Key Risks

| Risk | Impact | Recommendation |
|---|---|---|
| DB CPU saturation | High | Query/index optimization + load test |
| DB connection pressure | High | Keep pool within connection budget |
| Kafka lag during spike | Medium | Validate recovery rate |
| Redis cache miss storm | Medium | Load shedding / TTL strategy |
| Downstream retry amplification | High | Retry budget + circuit breaker |
| Non-linear app scaling | Medium | Scalability test 2/4/8/11 pods |

---

## 23. Validation Plan

Sizing là hypothesis, phải validate.

| Test | Mục tiêu |
|---|---|
| Load Test | Validate 2,000 TPS |
| Stress Test | Find breaking point |
| Scalability Test | Validate 10-11 pods |
| Soak Test | Validate memory/connections |
| Failover Test | Validate N-1 |
| Kafka Recovery Test | Validate backlog recovery |

**Graph cần có:** Có trong báo cáo test tương ứng, không cần lặp lại toàn bộ trong sizing report.

---

## 24. Final Conclusion

Ví dụ:

```text
Dựa trên current peak 1,000 TPS, growth 50% và safety margin 30%,
capacity mục tiêu được xác định khoảng 2,000 TPS.

Application layer cần khoảng 8 pods theo capacity lý thuyết,
khuyến nghị 10-11 pods để duy trì headroom.

Database dự kiến nhận khoảng 10,000 QPS và được xác định là
shared resource có rủi ro trở thành bottleneck đầu tiên.

Kafka cần tối thiểu 4 active consumers; đề xuất 6 partitions
để có thêm parallelism/headroom.

Sizing này chỉ được coi là hoàn tất sau khi validate bằng:
- Load Test
- Stress Test
- Scalability Test
- Soak Test
- N-1 Failover Test
```

---

# 25. Checklist Graph trong Sizing Report

| Graph | Mức độ | Mục đích |
|---|---|---|
| Peak TPS / Forecast / Target Capacity | Nên có | Giải thích capacity planning |
| TPS vs Pod Count | Nên có | Chứng minh scaling efficiency |
| DB QPS / CPU vs Load | Nên có nếu DB risk cao | Chứng minh DB capacity |
| Kafka Producer vs Consumer | Tùy trường hợp | Chứng minh consumer sizing |
| Storage Growth | Tùy trường hợp | Forecast dung lượng |
| Cost vs Capacity | Tùy trường hợp | Architecture/cost review |
| N-1 Capacity | Không cần graph | Bảng scenario thường đủ |

---

# 26. Nguyên tắc trình bày

Report sizing nên đi theo:

```text
Business Input
-> Calculation
-> Proposed Capacity
-> Risk
-> Validation
```

Không nên chỉ ghi:

```text
Cần 10 pods
```

mà phải chứng minh:

```text
Target TPS
/
Measured TPS per pod
+
Headroom
=
Recommended pod count
```

Sizing tốt là sizing:

- có input rõ,
- có công thức,
- có assumption,
- có risk,
- và được validate bằng load test thực tế.
