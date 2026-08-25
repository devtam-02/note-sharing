# Performance Testing & Bottleneck Analysis
## Từ Backend Developer đến Solution Architect

> Tài liệu này tập trung vào cách **thiết kế performance test, đọc kết quả, phát hiện bottleneck và tối ưu hệ thống**.  
> Nội dung đi từ nền tảng đến nâng cao, nhưng được tổ chức theo workflow thực tế thay vì chia cứng thành “basic/advanced”.

---

# 1. Performance Testing là gì?

Performance Testing trả lời các câu hỏi:

- Hệ thống xử lý được bao nhiêu tải?
- Latency ở tải bình thường và tải peak là bao nhiêu?
- Khi tăng tải thì component nào nghẽn trước?
- Hệ thống fail như thế nào khi vượt capacity?
- Sau spike/outage hệ thống có recovery được không?
- Có memory leak, connection leak hay performance degradation theo thời gian không?
- Khi scale thêm resource, throughput tăng được bao nhiêu?

Performance testing không chỉ đo API response time. Với hệ thống distributed/event-driven, phải đánh giá **end-to-end business flow**.

---

# 2. Performance Testing vs Performance Engineering

## Performance Testing

Chủ yếu trả lời:

```text
Hệ thống hiện tại chạy được bao nhiêu?
```

## Performance Engineering

Rộng hơn:

```text
Hệ thống cần đạt bao nhiêu?
Bottleneck nằm ở đâu?
Vì sao chưa đạt?
Scale như thế nào?
Trade-off kiến trúc là gì?
Chi phí để đạt target là bao nhiêu?
Failover/degradation thế nào?
```

Tư duy Dev thường tập trung vào:

```text
Method / Query / Service này có nhanh không?
```

Tư duy SA cần mở rộng thành:

```text
Toàn bộ business flow có đạt throughput, latency, availability,
scalability và cost target không?
```

---

# 3. Các loại Performance Test

## 3.1 Baseline Test

Chạy workload thấp, ổn định để lấy số liệu chuẩn.

Mục tiêu:

- biết latency bình thường,
- biết CPU/memory bình thường,
- có mốc so sánh sau tối ưu.

---

## 3.2 Load Test

Test ở tải production dự kiến hoặc peak hợp lý.

Mục tiêu:

```text
Ở target workload, hệ thống có giữ được SLO không?
```

Ví dụ:

```text
Target = 2,000 TPS
P95 < 400ms
P99 < 800ms
Error < 0.1%
```

---

## 3.3 Stress Test

Tăng tải vượt normal/peak để tìm:

- saturation point,
- breaking point,
- failure mode,
- khả năng recovery.

Ví dụ:

```text
500 -> 1,000 -> 2,000 -> 3,000 -> 5,000 TPS
```

---

## 3.4 Spike Test

Tăng tải đột ngột.

Dùng để kiểm tra:

- traffic burst,
- marketing campaign,
- flash sale,
- connection storm,
- autoscaling delay,
- cache miss storm,
- Kafka backlog.

---

## 3.5 Soak / Endurance Test

Chạy tải trong thời gian dài.

Tìm:

- memory leak,
- thread leak,
- connection leak,
- GC degradation,
- resource accumulation,
- log/disk growth,
- performance decay.

---

## 3.6 Scalability Test

Tăng resource rồi đo throughput.

Ví dụ:

```text
2 pods -> ?
4 pods -> ?
8 pods -> ?
```

Mục tiêu:

- xác định scaling efficiency,
- tìm shared bottleneck,
- xem throughput có scale gần tuyến tính hay không.

---

## 3.7 Capacity Test

Tìm mức tải tối đa mà hệ thống vẫn đạt SLO.

Đây mới là **usable capacity**.

Không nên gọi một mức TPS là capacity nếu:

```text
CPU = 100%
P99 = 10s
Error = 5%
```

---

## 3.8 Failover / Resilience Performance Test

Test performance khi:

- một pod chết,
- một node Kafka chết,
- Redis failover,
- DB replica fail,
- một AZ bị mất,
- downstream chậm/timeout.

SA cần quan tâm không chỉ steady-state capacity mà cả:

```text
Capacity during failure
```

---

# 4. Các khái niệm workload cơ bản

## 4.1 VU - Virtual User

User giả lập bởi load testing tool.

VU không phải TPS.

Ví dụ:

```text
100 VU
```

không đồng nghĩa:

```text
100 TPS
```

TPS phụ thuộc:

- response time,
- think time,
- số request mỗi flow,
- cách tool tạo workload.

---

## 4.2 Concurrency

Số request/work đang active đồng thời.

Ví dụ:

```text
500 requests đang chờ/được xử lý
```

Concurrency cao sẽ tác động tới:

- thread pool,
- DB connection pool,
- memory,
- downstream connections.

---

## 4.3 RPS / TPS

**RPS**: Requests per second.

**TPS**: Transactions per second.

Một business transaction có thể tạo nhiều internal requests.

Ví dụ:

```text
1 Order TPS
  -> 1 Promotion call
  -> 1 Inventory call
  -> 1 Payment call
```

Nếu:

```text
1,000 Order TPS
```

internal downstream traffic có thể >3,000 RPS.

---

## 4.4 Arrival Rate

Tốc độ request/work đến hệ thống.

Ví dụ:

```text
2,000 requests/s
```

Arrival rate rất quan trọng khi test backend system.

---

# 5. Closed workload vs Open workload

## Closed Model

Flow:

```text
User send request
-> wait response
-> think time
-> send next request
```

Nếu hệ thống chậm, load generator cũng gửi request chậm hơn.

Điều này có thể che bottleneck.

---

## Open Model

Request được tạo theo arrival rate độc lập với response time.

Ví dụ:

```text
1,000 arrivals/s
```

Dù system chậm, tool vẫn tạo workload gần 1,000 request/s.

Phù hợp hơn với:

- API backend,
- event traffic,
- Kafka producer,
- service-to-service load.

---

# 6. Coordinated Omission

Đây là lỗi đo latency khá nguy hiểm.

Ví dụ hệ thống pause 10 giây.

Nếu load generator chờ request hiện tại xong mới gửi tiếp, các request đáng lẽ phải đến trong 10 giây đó không được ghi nhận.

Kết quả:

```text
Latency report đẹp hơn thực tế.
```

Khi benchmark latency nghiêm túc, cần hiểu tool có xử lý vấn đề này hay không.

---

# 7. Latency metrics

Không chỉ nhìn Average.

Các chỉ số chính:

```text
P50
P90
P95
P99
Max
```

Ví dụ:

```text
P50 = 100ms
P95 = 400ms
P99 = 2.5s
Average = 180ms
```

Average nhìn đẹp nhưng 1% request cuối vẫn rất chậm.

---

# 8. Vì sao P95/P99 quan trọng?

Ví dụ:

```text
100,000 request/minute
```

1% tương ứng:

```text
1,000 requests/minute
```

Do đó P99 kém vẫn có thể ảnh hưởng lượng lớn user.

Đặc biệt quan trọng với:

- payment,
- login,
- checkout,
- realtime API,
- high-volume systems.

---

# 9. Throughput và Saturation

Pattern bình thường:

```text
Load tăng
TPS tăng
Latency ổn định
```

Đến một điểm:

```text
Load tăng
TPS gần như không tăng
Latency tăng mạnh
Error tăng
Queue tăng
```

Đây là dấu hiệu saturation.

Ví dụ:

| Offered Load | Achieved TPS | P95 | Error |
|---:|---:|---:|---:|
| 500 | 495 | 100ms | 0% |
| 1,000 | 990 | 150ms | 0% |
| 1,500 | 1,470 | 230ms | 0.05% |
| 2,000 | 1,900 | 500ms | 0.2% |
| 2,500 | 1,950 | 1.8s | 3% |

Breaking point nằm đâu đó trong vùng 2,000–2,500 TPS.

---

# 10. Utilization, Saturation, Errors - USE Method

Với mỗi resource hãy hỏi:

```text
U = Utilization
S = Saturation
E = Errors
```

Ví dụ DB pool:

```text
Utilization:
active / max

Saturation:
pending threads

Errors:
connection timeout
```

Áp dụng cho:

- CPU,
- memory,
- disk,
- network,
- thread pool,
- DB connection pool,
- Redis,
- Kafka.

---

# 11. RED Method cho service

Với API/service:

```text
R = Rate
E = Errors
D = Duration
```

Dashboard service tối thiểu:

```text
RPS/TPS
Error Rate
P50
P95
P99
```

Sau đó correlate với resource metrics.

---

# 12. Queueing: Service Time vs Wait Time

Total response time không chỉ là thời gian code chạy.

```text
Response Time
=
Wait Time
+
Service Time
+
Network/other overhead
```

Ví dụ:

```text
DB query execution = 30ms
Wait DB connection = 800ms
```

API có thể mất >830ms.

Root cause lúc này là **connection pool saturation**, không phải query execution time.

---

# 13. Little's Law

```text
Concurrency ≈ Throughput x Response Time
```

Ví dụ:

```text
TPS = 1,000
Average latency = 200ms = 0.2s
```

Concurrency:

```text
1,000 x 0.2
= 200
```

Nếu latency tăng lên 2s với cùng arrival rate:

```text
Concurrency ≈ 2,000
```

Điều này giải thích vì sao downstream chậm có thể kéo theo:

- thread tăng,
- connection tăng,
- memory tăng,
- queue tăng,
- cascading failure.

---

# 14. Load profile tốt nên có gì?

Ví dụ:

```text
Warm-up
-> Baseline
-> Ramp-up
-> Steady state
-> Peak
-> Spike
-> Recovery
-> Cool-down
```

Một test profile:

```text
0-5m    warm-up
5-15m   500 TPS
15-25m  1,000 TPS
25-35m  2,000 TPS
35-45m  3,000 TPS
45-50m  spike 5,000 TPS
50-60m  recover về 1,000 TPS
```

Không chỉ quan sát lúc tải tăng.

Phải xem recovery:

```text
Lag có giảm không?
CPU có trở về baseline không?
Memory có release không?
Connection có trở về normal không?
```

---

# 15. Workload model phải giống production

Không nên chỉ test:

```text
1 endpoint
1 user
1 data set
```

Production thường là mix:

```text
40% read
20% create
15% update
10% search
10% async event
5% report
```

Workload mix tạo contention thực trên:

- DB,
- Redis,
- CPU,
- Kafka,
- thread pool,
- locks.

---

# 16. Test data phải realistic

Ví dụ:

```text
Production DB = 100M rows
Test DB       = 10K rows
```

Kết quả có thể sai vì:

- toàn bộ data nằm trong memory,
- index rất nhỏ,
- cardinality khác,
- không có fragmentation,
- query plan khác.

Nên gần production về:

```text
Data volume
Cardinality
Distribution
Hot keys
Payload size
Retention
```

---

# 17. Warm-up

Đặc biệt với Java:

```text
JIT
Class loading
Connection pool
DB buffer
Redis cache
HTTP connection reuse
```

cần thời gian ổn định.

Không nên lấy số liệu startup làm baseline.

---

# 18. Dashboard metric tối thiểu

## Application

```text
RPS/TPS
P50/P95/P99
Success rate
Error rate
Timeout
Active requests
```

## JVM

```text
CPU
Heap
Heap after GC
Allocation rate
GC frequency
GC pause
Threads
Metaspace
Native memory
```

## Thread Pool

```text
Active
Max
Queue
Rejected
```

## DB

```text
Query latency
Slow queries
CPU
Connections
Pending connections
Lock wait
Deadlock
Rows scanned
IOPS
Replication lag
```

## Redis

```text
Latency
Hit ratio
Memory
CPU
Evictions
Hot keys
Big keys
Blocked clients
```

## Kafka

```text
Producer rate
Consumer rate
Lag
Lag growth
Recovery time
Rebalance
Retry
DLQ
Broker health
```

## Infrastructure

```text
Pod CPU
Pod memory
Restart
OOMKilled
Node utilization
Disk
Network
```

---

# 19. Phương pháp tìm bottleneck

Đi theo flow:

```text
Client
  |
Gateway / LB
  |
Application
  |
Thread pool
  |
Connection pool
  |
Database
  |
Redis
  |
Kafka
  |
Downstream service
```

Hỏi lần lượt:

1. Offered load là bao nhiêu?
2. Actual throughput là bao nhiêu?
3. P95/P99 bắt đầu tăng ở mức nào?
4. Queue xuất hiện ở đâu?
5. Resource nào gần saturation?
6. Error/retry xuất hiện ở đâu?
7. Bottleneck có phải shared resource không?
8. Scale component hiện tại có giúp không?

---

# 20. CPU bottleneck

## Dấu hiệu

```text
CPU > 90%
P95/P99 tăng
DB/downstream bình thường
```

## Kiểm tra

- hot methods,
- algorithm complexity,
- loop lớn,
- JSON serialization,
- regex,
- crypto,
- compression,
- logging,
- allocation,
- GC,
- lock contention.

## Công cụ

- JFR,
- async-profiler,
- flame graph,
- APM profiler,
- thread dump.

## Tối ưu

- giảm complexity,
- batch,
- cache computation,
- giảm payload,
- giảm serialization,
- giảm object allocation,
- tune thread/concurrency,
- scale CPU sau khi hiểu root cause.

---

# 21. Memory / GC bottleneck

## Dấu hiệu

```text
Heap tăng
GC thường xuyên
GC pause cao
P99 spike
OOM
Container restart
```

## Kiểm tra

- heap dump,
- allocation profile,
- large collections,
- local cache,
- ThreadLocal,
- static references,
- direct buffers,
- large payloads.

## Tối ưu

- giảm retained objects,
- bound cache,
- pagination,
- streaming,
- giảm allocation,
- fix leak,
- tune heap/GC sau khi hiểu allocation pattern.

---

# 22. Thread Pool saturation

## Dấu hiệu

```text
CPU không cao
Active thread = max
Queue tăng
Latency tăng
```

## Kiểm tra

- thread dump,
- blocked/waiting threads,
- DB calls,
- HTTP calls,
- locks,
- `Future.get`,
- synchronized sections.

## Tối ưu

- timeout,
- giảm blocking,
- bulkhead,
- điều chỉnh pool,
- async/reactive nếu đúng use case,
- giảm downstream latency.

Không tăng thread pool vô hạn.

---

# 23. DB Connection Pool saturation

## Dấu hiệu

```text
active = max
pending tăng
connection timeout
CPU app thấp
P99 cao
```

## Nguyên nhân

- query chậm,
- transaction dài,
- N+1,
- DB overload,
- connection leak,
- pool quá nhỏ.

## Tối ưu

- query/index,
- giảm transaction scope,
- batching,
- fix leak,
- sizing pool dựa trên DB capacity.

Pool lớn hơn không tự động tốt hơn.

---

# 24. Database bottleneck

## Dấu hiệu

```text
DB CPU cao
Query latency cao
Rows scanned lớn
Lock wait
```

## Kiểm tra

- EXPLAIN,
- slow query log,
- missing index,
- composite index,
- N+1,
- sort/group,
- pagination,
- transaction duration,
- lock contention.

## Tối ưu

- rewrite query,
- index,
- covering index,
- projection,
- batch query,
- keyset pagination,
- cache,
- read replica,
- partitioning,
- sharding khi thật sự cần.

---

# 25. Lock contention

## Dấu hiệu

```text
DB CPU không cao
TPS không tăng
Lock wait cao
Deadlock
```

## Tối ưu

- transaction ngắn,
- consistent lock ordering,
- optimistic locking,
- atomic update,
- tránh hot row,
- partition workload,
- retry deadlock có kiểm soát.

---

# 26. Redis bottleneck

## Dấu hiệu

```text
Redis latency tăng
CPU cao
Memory cao
Eviction tăng
Hit ratio giảm
```

## Kiểm tra

- hot key,
- big key,
- slow command,
- value size,
- serialization,
- fragmentation,
- network.

## Tối ưu

- split hot key,
- shard,
- TTL jitter,
- giảm value size,
- đúng data structure,
- local cache phù hợp,
- Redis Cluster,
- tránh command đắt.

---

# 27. Cache Stampede / Penetration / Avalanche

## Stampede

Nhiều request miss cùng key.

Giải pháp:

- single-flight,
- distributed lock,
- TTL jitter,
- refresh-ahead,
- stale-while-revalidate.

## Penetration

Request liên tục tới key không tồn tại.

Giải pháp:

- cache null,
- validation,
- bloom filter nếu phù hợp.

## Avalanche

Nhiều key hết hạn cùng lúc hoặc cache outage.

Giải pháp:

- TTL jitter,
- multi-level cache,
- rate limit,
- circuit breaker,
- warm-up.

---

# 28. Kafka lag bottleneck

## Dấu hiệu

```text
Producer rate > Consumer rate
Lag tăng liên tục
```

## Kiểm tra

- partition count,
- active consumer count,
- processing latency,
- DB/API dependency,
- retry,
- rebalance,
- batch size.

## Tối ưu

- tối ưu consumer,
- batching,
- scale consumer,
- tăng partition nếu cần,
- giảm downstream latency,
- DLQ/retry hợp lý.

Lưu ý:

```text
Active consumers <= partitions
```

trong cùng consumer group.

---

# 29. Kafka Rebalance

## Dấu hiệu

```text
Lag spike
Consumer pause
Rebalance thường xuyên
```

## Kiểm tra

- processing time,
- max.poll.interval.ms,
- restart/churn,
- session timeout,
- deployment pattern.

## Tối ưu

- giảm processing time,
- batch hợp lý,
- stable membership,
- tune consumer config,
- tránh restart không cần thiết.

---

# 30. External service bottleneck

## Dấu hiệu

```text
CPU thấp
Thread waiting cao
Downstream P95/P99 cao
```

## Tối ưu

- strict timeout,
- circuit breaker,
- bulkhead,
- retry có backoff+jitter,
- cache,
- async flow,
- fallback nếu business cho phép.

---

# 31. Retry Storm

Flow:

```text
Downstream chậm
-> timeout
-> retry
-> downstream càng quá tải
-> timeout nhiều hơn
-> retry nhiều hơn
```

Giải pháp:

- timeout nhỏ hợp lý,
- retry limit,
- exponential backoff,
- jitter,
- retry budget,
- circuit breaker,
- load shedding.

---

# 32. Network / Payload bottleneck

## Dấu hiệu

```text
CPU/DB ổn
Network cao
Latency tăng theo payload
```

## Kiểm tra

- request size,
- response size,
- Kafka message size,
- compression,
- serialization,
- cross-zone traffic.

## Tối ưu

- giảm payload,
- projection,
- pagination,
- batching,
- binary protocol khi phù hợp,
- compression có chọn lọc.

---

# 33. Async flow: API nhanh chưa chắc business nhanh

Ví dụ:

```text
POST /campaign = 120ms
Kafka lag = 2M
Business completion = 5 phút
```

API latency đẹp nhưng business SLO fail.

Cần đo:

```text
End-to-end latency
=
time final business state
-
time initial request/event
```

---

# 34. Latency Budget

Nếu:

```text
P95 end-to-end < 500ms
```

có thể chia:

```text
Gateway              20ms
Service A            80ms
DB                  100ms
Service B           100ms
Redis                20ms
Network/serialize    80ms
Buffer              100ms
```

Latency budget giúp thiết kế:

- timeout,
- retry,
- parallel call,
- cache,
- service boundaries.

---

# 35. Timeout Budget

Không hợp lý:

```text
Gateway timeout = 30s
Service A->B    = 30s
B->DB           = 30s
```

nếu SLO end-to-end chỉ 2s.

Timeout phải fit trong overall latency budget.

---

# 36. Amdahl's Law - tư duy tối ưu

Nếu:

```text
DB chiếm 80% latency
App logic chiếm 20%
```

tối ưu app logic nhanh 10 lần vẫn không làm toàn hệ thống nhanh 10 lần.

Nguyên tắc:

```text
Optimize the dominant cost first.
```

---

# 37. Shared Bottleneck và Scaling Efficiency

Ví dụ:

```text
2 pods -> 1,000 TPS
4 pods -> 1,800 TPS
8 pods -> 2,100 TPS
```

Scale càng nhiều, efficiency càng giảm.

Nguyên nhân thường:

- shared DB,
- shared lock,
- hot key,
- Kafka partitions,
- external service,
- network.

---

# 38. Load shedding

Khi hệ thống gần saturation, đôi khi tốt hơn là từ chối sớm một phần request thay vì để toàn hệ thống collapse.

Các kỹ thuật:

- rate limiting,
- bounded queues,
- reject fast,
- concurrency limit,
- admission control.

Tư duy SA:

```text
Controlled degradation
>
Uncontrolled collapse
```

---

# 39. Backpressure

Backpressure là cơ chế để downstream báo hoặc ép upstream giảm tốc độ.

Áp dụng trong:

- reactive streams,
- Kafka consumer pipeline,
- queue workers,
- batch processing.

Không có backpressure, producer nhanh hơn consumer có thể gây:

- memory growth,
- queue explosion,
- lag lớn,
- timeout cascade.

---

# 40. Failover performance

Một hệ thống có thể chịu 5,000 TPS ở normal state nhưng chỉ 2,500 TPS khi mất một AZ.

SA cần biết:

```text
Normal capacity
Failure capacity
Recovery time
```

Test nên bao gồm:

- kill pod,
- kill node,
- disable replica,
- downstream latency injection.

---

# 41. Autoscaling test

Không chỉ hỏi HPA có scale được không.

Cần đo:

```text
Detection delay
Scale-up delay
Pod startup time
Warm-up time
Traffic redistribution
Scale-down behavior
```

Nếu spike chỉ kéo dài 2 phút nhưng pod mất 3 phút để ready, HPA có thể không cứu được spike đó.

---

# 42. Performance Regression

Workflow:

```text
Baseline
-> Change
-> Same workload
-> Compare
```

Giữ gần như cố định:

- environment,
- data,
- load profile,
- duration,
- config.

Compare:

```text
TPS
P95/P99
CPU
Memory
GC
DB
Redis
Kafka
Error
Cost
```

---

# 43. Pass/Fail Criteria

Ví dụ:

```text
Workload = 3,000 TPS

PASS:
P95 < 400ms
P99 < 900ms
Error < 0.1%
CPU < 75%
Memory stable
DB pending ~= 0
Kafka lag không tăng liên tục
No OOM/restart
Recovery < 5 min sau spike
```

---

# 44. Anti-pattern khi test

- Chỉ nhìn average latency.
- Chỉ nhìn CPU/RAM.
- Test DB quá nhỏ.
- Không warm-up.
- Dùng một data key cho tất cả request.
- Không control arrival rate.
- Test quá ngắn.
- Không đo async flow.
- Không lưu baseline.
- Không monitor downstream.
- Chỉ scale app.
- Không test recovery/failover.
- Dùng max TPS làm capacity dù SLO đã fail.

---

# 45. Workflow Performance Engineering

```text
1. Define business SLO
2. Define workload model
3. Prepare realistic data
4. Baseline
5. Load test
6. Find saturation
7. Correlate metrics/traces
8. Identify root cause
9. Optimize one major bottleneck
10. Re-test same workload
11. Stress test
12. Soak test
13. Failover test
14. Validate recovery
15. Document sustainable capacity
```

---

# 46. Kỹ năng cần đạt khi đi từ Dev -> SA

## Dev

Cần đọc được:

- CPU,
- memory,
- GC,
- thread dump,
- query plan,
- pool metrics,
- Kafka lag,
- Redis metrics.

## Senior Dev / Tech Lead

Cần hiểu:

- workload model,
- saturation,
- queueing,
- concurrency,
- retry amplification,
- bulkhead,
- backpressure,
- end-to-end latency.

## Solution Architect

Cần trả lời được:

```text
Target capacity là gì?
Bottleneck ở đâu?
Shared resource nào giới hạn scale?
Failure capacity là bao nhiêu?
Headroom bao nhiêu?
Scaling efficiency thế nào?
Chi phí tăng ra sao?
Có cần thay đổi sync -> async?
Có cần cache/read replica/partition/sharding?
SLO và architecture có phù hợp nhau không?
```

---

# 47. Checklist phân tích một kết quả test

```text
[ ] Offered load?
[ ] Actual throughput?
[ ] P50/P95/P99?
[ ] Error rate?
[ ] Saturation bắt đầu ở mức nào?
[ ] CPU?
[ ] Memory/GC?
[ ] Thread pool queue?
[ ] DB pool pending?
[ ] Slow query?
[ ] Lock wait?
[ ] Redis hit ratio?
[ ] Kafka lag?
[ ] Downstream latency?
[ ] Retry count?
[ ] Scaling efficiency?
[ ] Recovery time?
[ ] Failure behavior?
[ ] End-to-end business latency?
```

---

# 48. Template kết luận

```text
Target workload:
3,000 TPS

Sustainable capacity:
2,600 TPS

Observed at 3,000 TPS:
- P95: 850ms
- P99: 2.1s
- Error: 1.2%
- App CPU: 58%
- DB CPU: 96%
- DB pending connections: high
- Kafka lag: stable

Primary bottleneck:
Database

Evidence:
- DB CPU saturation
- Query latency tăng
- DB pool pending tăng
- Application CPU chưa saturation

Optimization direction:
1. Slow query analysis
2. Composite/covering index
3. N+1 review
4. Transaction scope
5. Cache read-heavy path
6. Re-test cùng workload

Next test:
Stress test sau tối ưu để xác định breaking point mới.
```

---

# 49. Nguyên tắc cốt lõi

```text
Measure first.
Find saturation.
Find the dominant bottleneck.
Optimize root cause.
Re-test.
```

Performance là bài toán end-to-end, không phải chỉ là “tăng CPU” hay “tăng pod”.
