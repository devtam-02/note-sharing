# Capacity Planning & System Sizing
## TPS, Concurrency, App/DB/Kafka/Redis Sizing từ Dev đến Solution Architect

> Tài liệu này tập trung vào **tính tải, capacity planning, sizing resource và đưa ra quyết định scale/architecture**.

---

# 1. Capacity Planning là gì?

Capacity Planning trả lời:

```text
Hệ thống phải chịu bao nhiêu tải?
Trong bao lâu?
Với SLO nào?
Cần bao nhiêu resource?
Headroom bao nhiêu?
Khi một component fail thì còn bao nhiêu capacity?
```

Sizing không phải chỉ tính CPU/RAM.

Phải tính toàn chuỗi:

```text
Traffic
-> App
-> Thread/Connection
-> DB
-> Cache
-> Kafka
-> Downstream
-> Network
```

---

# 2. Bắt đầu từ business numbers

Các input nên có:

```text
Transactions/day
Peak hour
Peak factor
Growth %
Read/write ratio
Payload size
Retention
Concurrent users
SLO
Availability target
```

Không có các input này thì sizing chỉ là phỏng đoán.

---

# 3. Tính Average TPS

Công thức:

```text
Average TPS
=
Transactions per day / 86,400
```

Ví dụ:

```text
10,000,000 transactions/day
```

Average TPS:

```text
10,000,000 / 86,400
≈ 116 TPS
```

Không sizing production bằng average TPS.

---

# 4. Peak TPS

Nếu:

```text
Average TPS = 116
Peak factor = 8
```

Peak:

```text
116 x 8
≈ 928 TPS
```

Peak factor nên lấy từ:

- production metrics,
- business forecast,
- historical campaign data,
- event calendar.

Không nên chọn ngẫu nhiên nếu có dữ liệu thật.

---

# 5. Growth Factor và Safety Margin

Công thức đơn giản:

```text
Required Capacity
=
Peak Traffic
x Growth Factor
x Safety Margin
```

Ví dụ:

```text
Current peak    = 2,000 TPS
Growth          = 50%
Safety margin   = 30%
```

```text
2,000 x 1.5 x 1.3
= 3,900 TPS
```

Target:

```text
~4,000 TPS
```

---

# 6. Headroom

Ví dụ:

```text
Current peak = 3,000 TPS
Tested sustainable capacity = 3,200 TPS
```

Headroom:

```text
(3,200 - 3,000) / 3,000
≈ 6.7%
```

Rất thấp.

Nếu muốn 30%:

```text
3,000 x 1.3
= 3,900 TPS
```

---

# 7. Sizing phải gắn với SLO

Không được kết luận:

```text
Capacity = 5,000 TPS
```

nếu ở 5,000 TPS:

```text
P95 = 5s
Error = 4%
```

Nên dùng:

```text
Sustainable Capacity
=
Maximum throughput while SLO is met
```

---

# 8. Little's Law

```text
Concurrency
≈
Throughput x Response Time
```

Ví dụ:

```text
TPS = 1,000
Avg latency = 0.2s
```

Concurrency:

```text
1,000 x 0.2 = 200
```

Nếu latency tăng:

```text
2s
```

Concurrency:

```text
~2,000
```

Đây là cơ sở để hiểu:

- thread pressure,
- connection pressure,
- in-flight memory,
- downstream concurrency.

---

# 9. Business TPS vs Internal RPS

Ví dụ:

```text
1 Order
-> 2 DB queries
-> 1 Redis call
-> 2 HTTP calls
-> 3 Kafka messages
```

Nếu:

```text
Order = 1,000 TPS
```

thì internal load có thể là:

```text
DB QPS      = 2,000+
Redis OPS   = 1,000+
HTTP RPS    = 2,000+
Kafka msg/s = 3,000+
```

Sizing phải propagate workload xuống từng dependency.

---

# 10. Sizing App Pods

Giả sử test cho biết:

```text
1 pod sustainable:
350 TPS
CPU = 70%
P95 = 300ms
Error < 0.1%
```

Target:

```text
3,000 TPS
```

Pods lý thuyết:

```text
3,000 / 350
≈ 8.57
```

=> 9 pods.

Thêm 30% headroom:

```text
9 x 1.3
≈ 12 pods
```

Nhưng phải scale test lại.

---

# 11. Không giả định linear scaling

Sai:

```text
5 pods = 1,000 TPS
=> 25 pods = 5,000 TPS
```

Vì shared bottleneck:

- DB,
- Redis,
- Kafka partitions,
- downstream,
- lock,
- network.

---

# 12. Scaling Efficiency

```text
Scaling Efficiency
=
Throughput Increase / Resource Increase
```

Ví dụ:

```text
2 pods -> 1,000 TPS
4 pods -> 1,800 TPS
```

Resource x2.

Throughput x1.8.

```text
Efficiency = 1.8 / 2 = 90%
```

Nếu:

```text
2 pods -> 1,000
8 pods -> 1,500
```

shared bottleneck đang giới hạn.

---

# 13. CPU Sizing

CPU sizing không chỉ nhìn utilization.

Cần xem:

```text
CPU per TPS
P95/P99
GC
Thread contention
Burst headroom
```

Một target vận hành thường muốn chừa headroom, ví dụ:

```text
Normal peak CPU ~60-70%
```

Không phải rule tuyệt đối.

---

# 14. CPU per transaction

Có thể theo dõi:

```text
CPU cost per 1,000 TPS
```

Ví dụ:

```text
1 pod, 2 cores
1,000 TPS
CPU = 50%
```

Nếu workload tương đối ổn định, metric này giúp dự báo scale.

Nhưng phải kiểm chứng vì system có thể non-linear khi gần saturation.

---

# 15. JVM / Memory Sizing

Container memory gồm:

```text
Heap
Metaspace
Thread stack
Direct buffer
Code cache
Native memory
Libraries
```

Nếu:

```yaml
memory limit: 3Gi
```

không nên mặc định:

```text
-Xmx3g
```

Cần chừa native headroom.

---

# 16. Memory sizing theo workload

Cần xem:

```text
Live heap
Peak heap
Allocation rate
Concurrency
Payload size
Buffering
Local cache
Thread count
```

Nếu mỗi in-flight request giữ:

```text
100KB
```

và concurrency:

```text
2,000
```

chỉ riêng request state có thể khoảng:

```text
200MB
```

chưa tính object overhead và component khác.

---

# 17. Thread Pool sizing

Approximation blocking workload:

```text
Threads
≈
CPU cores x (1 + Wait Time / Compute Time)
```

Ví dụ:

```text
Compute = 20ms
Wait I/O = 80ms
CPU cores = 4
```

```text
Threads ≈ 4 x (1 + 80/20)
= 20
```

Chỉ là điểm bắt đầu.

Phải benchmark.

---

# 18. Thread quá nhiều có gì xấu?

- context switching,
- memory stack,
- downstream overload,
- lock contention,
- scheduling overhead.

Sizing phải theo entire dependency chain.

---

# 19. DB Connection Pool Budget

Ví dụ:

```text
20 pods
maxPoolSize = 50
```

Theoretical:

```text
1,000 DB connections
```

Nếu DB:

```text
max_connections = 500
```

rõ ràng không ổn.

---

# 20. Chia connection budget

Giả sử:

```text
DB safe connection budget = 300
Pods = 10
```

Sơ bộ:

```text
300 / 10
= 30/pod
```

Nhưng cần chừa cho:

- service khác,
- admin,
- migration,
- monitoring,
- failover.

---

# 21. Pool size không phải càng lớn càng tốt

Nếu DB đang chậm:

```text
pool 50 -> pool 200
```

có thể làm DB tệ hơn vì contention tăng.

Phải tìm:

- query latency,
- active/pending,
- DB CPU,
- lock wait,
- connection hold time.

---

# 22. DB QPS Sizing

Ví dụ:

```text
API = 2,000 TPS
Average DB queries/request = 5
```

Theoretical DB QPS:

```text
10,000 queries/s
```

Nếu có cache hit 80% cho 2 trong 5 query, DB QPS thực có thể giảm đáng kể.

Phải model theo actual flow.

---

# 23. DB Storage Sizing

Input:

```text
rows/day
average row size
index overhead
retention
replication
growth
```

Ví dụ:

```text
5M rows/day
avg row = 500 bytes
```

Raw:

```text
2.5GB/day
```

Nếu index + overhead ~80%:

```text
~4.5GB/day
```

90 ngày:

```text
~405GB
```

Chưa tính:

- binary logs,
- temp space,
- replication,
- backups,
- safety margin.

---

# 24. IOPS / Disk Sizing

DB performance có thể bị giới hạn bởi storage.

Cần xem:

- random read/write,
- sequential throughput,
- fsync rate,
- WAL/binlog,
- checkpoint,
- compaction.

Không thể sizing DB chỉ bằng CPU/RAM.

---

# 25. Read Replica

Phù hợp khi:

```text
Read-heavy
Eventual read consistency chấp nhận được
```

Giúp offload:

- report,
- search,
- read APIs.

Không giải quyết write bottleneck.

---

# 26. Partitioning

Dùng để:

- giảm data scanned,
- quản lý retention,
- maintenance,
- chia dữ liệu theo time/range/key.

Partitioning không tự động tăng write capacity across machines như sharding.

---

# 27. Sharding

Xem xét khi:

```text
Single DB không còn đủ capacity
Data/traffic có shard key tốt
Operational complexity chấp nhận được
```

Phải đánh giá:

- hot shard,
- cross-shard query,
- cross-shard transaction,
- resharding,
- global uniqueness.

---

# 28. Cache Sizing

Sơ bộ:

```text
Memory
≈
Number of keys
x
Average key/value size
```

Ví dụ:

```text
10M keys
1KB/value
```

Raw:

```text
~10GB
```

Thực tế cần cộng:

- key size,
- object overhead,
- allocator fragmentation,
- replication,
- data structure overhead,
- safety margin.

---

# 29. Cache Hit Ratio và DB Load

Traffic:

```text
10,000 RPS
```

Hit 95%:

```text
DB load = 500 RPS
```

Hit 80%:

```text
DB load = 2,000 RPS
```

DB load tăng 4x dù traffic user không đổi.

---

# 30. Cache capacity phải tính failure mode

Nếu Redis down:

```text
100% request -> DB
```

Câu hỏi SA phải trả lời:

```text
DB có chịu nổi cache-miss storm không?
```

Nếu không:

- rate limit,
- graceful degradation,
- stale cache,
- local cache,
- load shedding.

---

# 31. Kafka Consumer Sizing

Ví dụ:

```text
Producer = 10,000 msg/s
1 consumer = 2,000 msg/s
```

Cần:

```text
5 active consumers
```

Nhưng:

```text
active consumers <= partition count
```

Nếu topic chỉ có 3 partitions:

```text
max active consumers = 3
```

---

# 32. Kafka Partition Sizing

Partition count cần cân bằng:

- throughput,
- parallelism,
- ordering,
- rebalance,
- broker resources,
- future growth.

Không phải càng nhiều càng tốt.

---

# 33. Kafka Lag Growth

```text
Lag Growth Rate
=
Producer Rate - Consumer Rate
```

Ví dụ:

```text
Producer = 5,000
Consumer = 4,200
```

Lag tăng:

```text
800 msg/s
```

10 phút:

```text
800 x 600
=
480,000 messages
```

---

# 34. Kafka Backlog Recovery

```text
Recovery Rate
=
Consumer Capacity - Producer Rate
```

Ví dụ:

```text
Lag = 1,000,000
Producer = 5,000/s
Consumer = 7,000/s
```

Recovery:

```text
2,000/s
```

Time:

```text
1,000,000 / 2,000
= 500s
≈ 8.3m
```

---

# 35. Sizing Kafka không chỉ theo msg/s

Phải tính thêm:

```text
Message size
Retention
Replication factor
Compression
Broker disk throughput
Network throughput
Consumer processing
```

Ví dụ:

```text
10,000 msg/s
avg message = 5KB
```

Raw ingress:

```text
~50MB/s
```

Replication factor 3 làm storage/network requirement lớn hơn đáng kể.

---

# 36. HTTP downstream amplification

Ví dụ:

```text
Service A = 2,000 RPS
A gọi B 3 lần/request
```

B nhận:

```text
6,000 RPS
```

Nếu retry 3 lần:

```text
worst-case ~18,000 calls/s
```

Sizing downstream phải tính fan-out và retry.

---

# 37. Retry Budget

Không nên cho retry vô hạn.

Có thể đặt:

```text
Max retries/request
Global retry rate
Timeout budget
```

Mục tiêu:

```text
Không để retry traffic trở thành load lớn hơn normal traffic.
```

---

# 38. Network Sizing

Tính sơ bộ:

```text
Bandwidth
≈
RPS x Payload Size
```

Ví dụ:

```text
5,000 RPS
response = 200KB
```

Outbound:

```text
~1GB/s
```

chưa tính:

- protocol overhead,
- retries,
- replication,
- service-to-service calls.

---

# 39. Serialization Cost

Payload lớn không chỉ tốn network.

Nó còn tốn:

- CPU serialize,
- CPU deserialize,
- memory buffer,
- GC,
- connection occupancy.

Giảm payload thường cải thiện nhiều layer cùng lúc.

---

# 40. HPA / Autoscaling Sizing

Không chỉ chọn:

```text
CPU 70%
```

Phải hiểu:

```text
Metric detection delay
Scale decision delay
Pod startup time
Warm-up
Readiness
Traffic rebalance
```

Nếu spike 2 phút nhưng pod cần 3 phút để ready, CPU HPA không giải quyết được spike.

---

# 41. Metric để autoscale

Có thể dùng:

- CPU,
- memory,
- RPS,
- queue depth,
- Kafka lag,
- active requests.

Metric tốt nên correlate với workload.

Ví dụ Kafka consumer:

```text
Lag / processing capacity
```

thường hữu ích hơn CPU đơn thuần.

---

# 42. Scale-up vs Scale-out

## Scale-up

Tăng:

- CPU,
- RAM,
- DB instance size.

Ưu:

- đơn giản.

Nhược:

- giới hạn vật lý,
- single-node dependency,
- cost curve.

## Scale-out

Tăng:

- pod,
- instance,
- consumer,
- replica.

Ưu:

- resilience,
- elasticity.

Nhược:

- coordination,
- shared bottleneck,
- distributed complexity.

---

# 43. Failure Capacity

Giả sử:

```text
Normal:
3 AZ
Total capacity = 6,000 TPS
```

Mất 1 AZ:

```text
Remaining theoretical = 4,000 TPS
```

Nếu business peak:

```text
5,000 TPS
```

hệ thống sẽ overload khi failover.

SA phải sizing theo:

```text
N-1 capacity
```

nếu availability requirement đòi hỏi.

---

# 44. N+1 Capacity Thinking

Với critical service, có thể hỏi:

```text
Nếu mất 1 pod?
Nếu mất 1 node?
Nếu mất 1 AZ?
Nếu DB failover?
```

Capacity planning cần tính redundancy.

---

# 45. Latency Budget

Nếu end-to-end P95 target:

```text
500ms
```

có thể chia:

```text
Gateway 20ms
Service A 80ms
DB 100ms
Service B 100ms
Network/serialization 80ms
Buffer 120ms
```

Latency budget ảnh hưởng đến:

- timeout,
- retry,
- synchronous fan-out,
- service decomposition.

---

# 46. Synchronous Fan-out

Flow:

```text
A
|- B
|- C
|- D
|- E
```

Nếu gọi tuần tự:

```text
latency cộng dồn
```

Nếu gọi parallel:

```text
latency giảm
nhưng concurrency/downstream load tăng
```

SA cần trade-off:

- dependency,
- latency SLO,
- connection budget,
- downstream capacity.

---

# 47. Async Capacity

Async giúp absorb burst nhưng không làm work biến mất.

Ví dụ:

```text
Producer peak = 10k/s
Consumer = 6k/s
```

Queue sẽ tăng.

Async chỉ chuyển bài toán:

```text
Immediate latency
->
Backlog + recovery capacity
```

---

# 48. End-to-End Capacity

Một approximation:

```text
System Capacity
≈
min(
 App capacity,
 DB capacity,
 Cache capacity,
 Kafka capacity,
 Consumer capacity,
 Downstream capacity,
 Network capacity
)
```

Đây là tư duy bottleneck law.

---

# 49. Capacity và Cost

SA nên tính:

```text
Cost per TPS
Cost per transaction
Cost per million events
```

Ví dụ:

```text
20 pods = $X/month
capacity = 4,000 TPS
```

Sau tối ưu:

```text
12 pods
same capacity
```

Performance optimization có thể là cost optimization.

---

# 50. Performance vs Cost Curve

Không phải target luôn là max performance.

Có thể có 3 option:

```text
Option A:
P95 200ms
Cost 100

Option B:
P95 300ms
Cost 60

Option C:
P95 500ms
Cost 40
```

Nếu SLO là 400ms, B có thể là lựa chọn tốt nhất.

---

# 51. Capacity Forecasting

Ngoài peak hiện tại cần forecast:

```text
3 months
6 months
12 months
```

Input:

- customer growth,
- transaction growth,
- campaign/event growth,
- data growth,
- new feature impact.

---

# 52. Seasonality

Traffic có thể thay đổi theo:

- giờ trong ngày,
- ngày trong tuần,
- payday,
- holiday,
- flash sale,
- campaign,
- month-end.

Peak factor nên dựa vào seasonality nếu có.

---

# 53. Burst Capacity

Average peak chưa đủ.

Ví dụ:

```text
Normal peak = 2,000 TPS
10-second burst = 8,000 TPS
```

Nếu system có queue/buffer tốt, có thể absorb burst.

Nếu toàn bộ sync, dễ collapse.

---

# 54. Queue Buffer Sizing

Nếu:

```text
Burst producer = 10k/s
Consumer = 6k/s
Burst duration = 60s
```

Backlog tạo ra:

```text
(10k - 6k) x 60
= 240k messages
```

Queue phải chịu được ít nhất backlog này cộng headroom.

---

# 55. Recovery SLO

Không chỉ có response SLO.

Có thể định nghĩa:

```text
Sau peak, lag phải về <10k trong 10 phút.
```

Đây là SLO tốt cho async system.

---

# 56. Data Growth Sizing

Ví dụ:

```text
Events/day = 20M
Avg event = 2KB
Retention = 7 days
Replication = 3
```

Raw:

```text
20M x 2KB
≈ 40GB/day
```

7 ngày:

```text
280GB
```

Replication 3:

```text
~840GB
```

Chưa tính overhead.

---

# 57. Redis Data Growth

Nếu:

```text
New keys/day = 1M
avg key+value = 800 bytes
TTL = 7 days
```

Approx active data:

```text
1M x 7 x 800 bytes
≈ 5.6GB raw
```

Cần cộng overhead/replication/headroom.

---

# 58. DB Growth và Index Growth

Index có thể chiếm tỷ lệ lớn.

Không sizing storage chỉ theo table rows.

Phải tính:

```text
Table
Indexes
WAL/binlog
Temp
Backups
Replica
Free space
```

---

# 59. Capacity Review Checklist

## Traffic

```text
[ ] Avg TPS
[ ] Peak TPS
[ ] Burst TPS
[ ] Growth
[ ] Seasonality
[ ] Workload mix
```

## App

```text
[ ] Sustainable TPS/pod
[ ] CPU/TPS
[ ] Memory/pod
[ ] Scaling efficiency
[ ] Startup/warm-up time
```

## DB

```text
[ ] QPS
[ ] Connection budget
[ ] CPU
[ ] IOPS
[ ] Storage growth
[ ] Lock contention
[ ] Read/write ratio
```

## Cache

```text
[ ] Hit ratio
[ ] Memory
[ ] Failure behavior
[ ] Hot keys
```

## Kafka

```text
[ ] msg/s
[ ] message size
[ ] partitions
[ ] consumer capacity
[ ] lag recovery
[ ] retention
```

## Downstream

```text
[ ] fan-out
[ ] timeout
[ ] retry
[ ] capacity contract
```

## Availability

```text
[ ] N-1 capacity
[ ] AZ failure
[ ] failover time
[ ] recovery capacity
```

---

# 60. Example tổng hợp

Input:

```text
Current peak          = 1,000 TPS
Growth                = 3x
Safety margin         = 30%
Sustainable pod       = 250 TPS
DB safe connections   = 300
Kafka producer        = 6,000 msg/s
Consumer capacity     = 1,500 msg/s
```

## Target

```text
1,000 x 3 x 1.3
= 3,900 TPS
```

Làm tròn:

```text
4,000 TPS
```

## App

```text
4,000 / 250
= 16 pods
```

## DB pool budget

```text
300 / 16
≈ 18 connections/pod
```

## Kafka

```text
6,000 / 1,500
= 4 active consumers
```

=> ít nhất 4 partitions, thực tế thêm headroom nếu cần.

---

# 61. Tư duy SA khi thấy target tăng 5x

Không trả lời ngay:

```text
Scale pod x5
```

Phải hỏi:

```text
Current per-pod sustainable TPS?
Scaling efficiency?
DB capacity?
DB connection budget?
Redis capacity?
Kafka partition limit?
Downstream capacity?
Network?
Failure capacity?
Cost?
```

---

# 62. Khi nào cần thay đổi architecture?

Sau khi tối ưu implementation mà vẫn không đạt target, xem xét:

## Sync -> Async

Khi:

- user không cần kết quả ngay,
- cần absorb burst,
- downstream nặng.

## Add Cache

Khi:

- read-heavy,
- repeated expensive query,
- consistency cho phép.

## Read Replica

Khi:

- read bottleneck.

## Partition

Khi:

- large table,
- time/range access,
- retention.

## Sharding

Khi:

- single DB không đáp ứng,
- shard key tốt,
- chấp nhận complexity.

## Event-driven

Khi:

- decoupling,
- buffering,
- eventual consistency phù hợp.

---

# 63. Khi nào không nên scale?

Không nên scale thêm app nếu:

```text
App CPU 40%
DB CPU 100%
```

Không nên tăng consumer nếu:

```text
partitions = 3
active consumers = 3
```

Không nên tăng DB pool nếu:

```text
DB đã saturation
```

Không nên thêm Redis nếu:

```text
hit ratio thấp vì key strategy sai
```

Tăng resource sai layer chỉ chuyển hoặc làm nặng bottleneck.

---

# 64. Capacity Decision Framework

```text
1. Business target
2. SLO
3. Peak/burst/growth
4. Propagate workload to dependencies
5. Baseline current capacity
6. Find bottleneck
7. Size each layer
8. Add headroom
9. Check N-1/failover capacity
10. Validate by load/stress test
11. Compare cost
12. Document limits
```

---

# 65. Output của một SA capacity review

Một tài liệu capacity review nên có:

```text
Traffic assumptions
SLO
Current capacity
Target capacity
Per-component sizing
Known bottlenecks
Headroom
Failure capacity
Scaling strategy
Cost implication
Test evidence
Risk
Next review date
```

---

# 66. Template kết luận sizing

```text
Business target:
4,000 TPS

SLO:
P95 < 400ms
P99 < 900ms
Error < 0.1%

Current sustainable:
2,600 TPS

Proposed:
- App: 16 pods
- DB pool: 18/pod
- Kafka: >= 4 active partitions/consumers
- Redis: sized for 2x active dataset
- Headroom: 30%

Critical risk:
Database CPU reaches saturation before app CPU.

Required optimization:
1. Index/query review
2. Cache high-read path
3. Reduce transaction duration

Validation:
Run 4,000 TPS steady-state
Run 5,000 TPS stress
Run 2-hour soak
Run N-1 failover test
```

---

# 67. Kỹ năng cần đạt khi từ Dev -> SA

## Dev

Hiểu:

- TPS/RPS,
- latency,
- CPU/memory,
- DB pool,
- Kafka lag.

## Senior Dev

Biết:

- workload propagation,
- Little's Law,
- per-pod capacity,
- connection sizing,
- Kafka partition/consumer sizing,
- cache effect.

## SA

Phải nhìn được:

```text
Business -> workload -> architecture -> capacity -> failure -> cost
```

và giải thích được vì sao chọn:

- scale-up,
- scale-out,
- cache,
- async,
- replica,
- partition,
- sharding.

---

# 68. Nguyên tắc cốt lõi

```text
Capacity is not maximum throughput.
Capacity is sustainable throughput under SLO.
```

Và:

```text
System capacity is limited by the weakest shared bottleneck.
```

Sizing chỉ đúng khi được **kiểm chứng bằng performance test**.
