# SYSTEM SIZING GUIDE
## Các bước tính sizing hệ thống từ Business Requirement đến Production Capacity

> Mục tiêu của tài liệu:
>
> - Chuyển business load thành technical capacity.
> - Tính TPS/RPS, concurrency, peak load và headroom.
> - Sizing sơ bộ Application, CPU, RAM, Thread Pool, DB, Redis, Kafka và Network.
> - Kiểm tra khả năng scale, failover và growth.
> - Đưa ra sizing có cơ sở thay vì chọn tài nguyên cảm tính.

---

# 1. Thu thập Business Input

Trước khi sizing, cần có các số liệu đầu vào.

| Thông số | Ví dụ |
|---|---:|
| Transactions/day | 10,000,000 |
| Current Peak TPS | 1,000 TPS |
| Expected Growth | 50% |
| Peak Factor | 8x average |
| Safety Margin | 30% |
| Read/Write Ratio | 70/30 |
| Payload Avg | 20 KB |
| Retention | 90 ngày |
| Availability | 99.9% |
| End-to-End P95 | < 500 ms |

Không có các thông số này, sizing chỉ là estimate rất thô.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Business Load | Tải phát sinh từ nghiệp vụ |
| Peak Load | Mức tải cao nhất |
| Growth Rate | Tỷ lệ tăng trưởng dự kiến |
| Safety Margin | Phần dự phòng |
| Retention | Thời gian giữ dữ liệu |
| Read/Write Ratio | Tỷ lệ đọc/ghi |

---

# 2. Tính Average TPS

Công thức:

```text
Average TPS
=
Total Transactions / Total Seconds
```

Ví dụ:

```text
10,000,000 transactions/day
```

Một ngày:

```text
86,400 seconds
```

```text
Average TPS
=
10,000,000 / 86,400
≈ 116 TPS
```

### Lưu ý

Không dùng Average TPS để sizing production nếu traffic có peak.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| TPS | Transactions Per Second |
| Average TPS | TPS trung bình |
| Transaction | Một đơn vị nghiệp vụ hoàn chỉnh |

---

# 3. Tính Peak TPS

Nếu có production metrics thì dùng peak thực tế.

Nếu chưa có, có thể estimate bằng Peak Factor.

Công thức:

```text
Peak TPS
=
Average TPS × Peak Factor
```

Ví dụ:

```text
Average TPS = 116
Peak Factor = 8
```

```text
Peak TPS
=
116 × 8
≈ 928 TPS
```

Nếu metrics cho thấy:

```text
Average = 100 TPS
Peak = 800 TPS
```

thì:

```text
Peak Factor
=
Peak / Average
=
8
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Peak TPS | TPS cao nhất cần xử lý |
| Peak Factor | Hệ số Peak / Average |
| Burst | Tải tăng mạnh trong thời gian ngắn |

---

# 4. Tính Target Capacity

Phải tính thêm growth và safety margin.

Công thức:

```text
Required Capacity
=
Peak TPS
× Growth Factor
× Safety Margin Factor
```

Trong đó:

```text
Growth Factor
=
1 + Growth Rate

Safety Margin Factor
=
1 + Safety Margin
```

Ví dụ:

```text
Peak TPS = 1,000
Growth = 50%
Safety Margin = 30%
```

```text
Growth Factor = 1.5
Safety Margin Factor = 1.3
```

```text
Required Capacity
=
1,000 × 1.5 × 1.3
=
1,950 TPS
```

Có thể làm tròn:

```text
Target Capacity ≈ 2,000 TPS
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Required Capacity | Capacity cần đạt |
| Growth Factor | Hệ số tăng trưởng |
| Safety Margin | Phần tải dự phòng |
| Headroom | Phần capacity còn dư |

---

# 5. Tính Workload theo từng API/Flow

Không phải mọi endpoint đều nhận cùng tải.

Ví dụ:

| Flow | Tỷ lệ |
|---|---:|
| Read Campaign | 40% |
| Validate | 20% |
| Cashback | 15% |
| Create | 10% |
| Search | 10% |
| Other | 5% |

Nếu:

```text
Total TPS = 2,000
```

Công thức:

```text
Flow TPS
=
Total TPS × Traffic Ratio
```

Ví dụ:

```text
Read Campaign
=
2,000 × 40%
=
800 TPS
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Workload Mix | Tỷ lệ các luồng |
| Traffic Ratio | Tỷ lệ traffic của từng flow |
| Flow TPS | TPS của một luồng cụ thể |

---

# 6. Propagate tải xuống các dependency

Một transaction business có thể tạo nhiều call nội bộ.

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
Order TPS = 2,000
```

thì approximate:

```text
DB QPS      = 4,000+
Redis OPS   = 2,000+
HTTP RPS    = 4,000+
Kafka msg/s = 6,000+
```

### Công thức

```text
Dependency Load
=
Business TPS × Calls per Transaction
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Fan-out | Một request gọi nhiều dependency |
| Internal RPS | Request nội bộ |
| DB QPS | Query Per Second |
| Redis OPS | Redis operations/sec |
| Kafka msg/s | Message mỗi giây |

---

# 7. Tính Concurrency bằng Little's Law

Công thức:

```text
Concurrency
≈
Throughput × Response Time
```

Ví dụ:

```text
TPS = 2,000
Average latency = 200ms = 0.2s
```

```text
Concurrency
=
2,000 × 0.2
=
400
```

Tức trung bình khoảng:

```text
400 requests in-flight
```

Nếu latency tăng lên 1s:

```text
Concurrency
=
2,000 × 1
=
2,000
```

### Ý nghĩa

Concurrency ảnh hưởng:

- thread pool,
- DB pool,
- memory,
- downstream connections.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Concurrency | Số request đồng thời |
| Throughput | Số request xử lý mỗi giây |
| Response Time | Thời gian xử lý |
| In-flight | Request đang được xử lý/chờ |

---

# 8. Tính số Pod / Instance

Cần benchmark để biết sustainable TPS/pod.

Ví dụ:

```text
1 pod:
Sustainable TPS = 350
CPU = 70%
P95 = 300ms
Error < 0.1%
```

Target:

```text
3,000 TPS
```

Công thức:

```text
Pods Required
=
Target TPS / Sustainable TPS per Pod
```

```text
3,000 / 350
≈ 8.57
```

=> tối thiểu:

```text
9 pods
```

Nếu muốn 30% headroom:

```text
9 × 1.3
≈ 12 pods
```

### Lưu ý

Không giả định:

```text
2x pods = 2x throughput
```

Phải kiểm tra shared bottleneck.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| TPS per Pod | Throughput bền vững của một pod |
| Horizontal Scale | Tăng số pod |
| Sustainable TPS | TPS vẫn giữ được SLO |
| Shared Bottleneck | DB/Redis/Kafka dùng chung |

---

# 9. Tính Scaling Efficiency

Ví dụ:

```text
2 pods = 1,000 TPS
4 pods = 1,800 TPS
```

Resource factor:

```text
4 / 2 = 2
```

Throughput factor:

```text
1,800 / 1,000 = 1.8
```

Công thức:

```text
Scaling Efficiency
=
Throughput Factor
/
Resource Factor
× 100%
```

```text
1.8 / 2 × 100%
=
90%
```

Nếu:

```text
2 pods = 1,000 TPS
8 pods = 1,500 TPS
```

=> scale app không còn hiệu quả.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Scaling Efficiency | Hiệu quả tăng throughput |
| Linear Scaling | Resource tăng bao nhiêu, throughput tăng gần tương ứng |
| Diminishing Return | Scale thêm nhưng lợi ích giảm |

---

# 10. CPU Sizing

Không sizing CPU chỉ bằng peak utilization.

Cần xem:

- CPU/TPS,
- P95/P99,
- GC,
- thread contention,
- throttling.

Ví dụ:

```text
1 pod
2 vCPU
1,000 TPS
CPU = 50%
```

Có thể dùng để estimate sơ bộ:

```text
2,000 TPS
≈ 4 vCPU equivalent
```

nhưng chỉ khi workload scale gần tuyến tính.

### Khuyến nghị

Production nên có headroom.

Ví dụ:

```text
Normal peak CPU ~60-70%
```

không phải rule cố định.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| CPU Utilization | % CPU đang dùng |
| CPU Throttling | Container bị giới hạn CPU |
| CPU-bound | Workload chủ yếu tốn CPU |

---

# 11. Memory / JVM Sizing

Container memory gồm:

```text
Heap
Metaspace
Thread Stack
Direct Buffer
Code Cache
Native Memory
```

Nếu:

```yaml
memory limit: 3Gi
```

không nên:

```text
-Xmx3g
```

vì JVM cần memory ngoài heap.

## 11.1. Tính memory theo concurrency

Ví dụ:

```text
Average in-flight request state = 100KB
Concurrency = 2,000
```

Approx:

```text
100KB × 2,000
=
200MB
```

chưa tính object overhead.

## 11.2. Theo dõi

```text
Heap used
Heap after GC
Allocation rate
GC pause
Native memory
RSS
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Heap | Memory chứa Java object |
| Metaspace | Metadata class |
| Direct Buffer | Buffer ngoài heap |
| RSS | Resident memory của process |
| GC | Garbage Collection |

---

# 12. Thread Pool Sizing

Với blocking workload, approximation:

```text
Threads
≈
CPU Cores × (1 + Wait Time / Compute Time)
```

Ví dụ:

```text
CPU = 4 cores
Compute = 20ms
Wait = 80ms
```

```text
Threads
≈
4 × (1 + 80/20)
=
20
```

Đây chỉ là initial estimate.

Phải benchmark.

### Thread quá nhỏ

```text
Queue tăng
Latency tăng
```

### Thread quá lớn

```text
Context switching
Memory tăng
Downstream overload
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Thread Pool | Nhóm thread xử lý |
| Wait Time | Thời gian chờ I/O |
| Compute Time | Thời gian CPU thực thi |
| Context Switching | CPU đổi giữa các thread |

---

# 13. DB Connection Pool Sizing

Ví dụ:

```text
20 pods
maxPoolSize = 50
```

Theoretical:

```text
20 × 50
=
1,000 connections
```

Nếu DB:

```text
safe max connections = 500
```

configuration không phù hợp.

## 13.1. Connection Budget

Giả sử:

```text
DB budget cho service = 300
Pods = 10
```

```text
Pool/pod
<=
300 / 10
=
30
```

Cần chừa connection cho:

- service khác,
- admin,
- migration,
- monitoring,
- failover.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| DB Connection Budget | Tổng connection dành cho service |
| maxPoolSize | Max connection/pod |
| Active Connection | Connection đang dùng |
| Pending | Request đang chờ connection |

---

# 14. DB QPS Sizing

Nếu:

```text
Application TPS = 2,000
Queries per transaction = 5
```

Approx:

```text
DB QPS
=
2,000 × 5
=
10,000 QPS
```

Phải tính thêm:

- cache,
- batch,
- N+1,
- retry,
- read replica.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| QPS | Queries Per Second |
| Query Amplification | Một request tạo nhiều query |
| N+1 | Một query chính kéo theo nhiều query phụ |

---

# 15. DB Storage Sizing

Input:

```text
Rows/day
Average row size
Index overhead
Retention
Replication
Growth
```

Ví dụ:

```text
5M rows/day
Average row = 500 bytes
```

Raw data/day:

```text
5,000,000 × 500
=
2.5GB/day
```

Nếu index + overhead ~80%:

```text
~4.5GB/day
```

90 ngày:

```text
4.5 × 90
=
405GB
```

Chưa tính:

- binlog/WAL,
- temp space,
- backup,
- replica,
- safety margin.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Row Size | Kích thước trung bình row |
| Index Overhead | Storage cho index |
| WAL/Binlog | Transaction/write log |
| Retention | Số ngày giữ data |

---

# 16. DB IOPS / Disk Sizing

DB có thể bottleneck vì storage.

Theo dõi:

- random read/write,
- sequential throughput,
- fsync,
- WAL/binlog,
- checkpoint,
- disk latency.

Sizing DB không chỉ dựa trên CPU/RAM.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| IOPS | IO operations per second |
| Disk Latency | Thời gian mỗi IO |
| fsync | Flush data xuống disk |
| Checkpoint | Ghi dirty pages xuống storage |

---

# 17. Redis Memory Sizing

Công thức sơ bộ:

```text
Redis Memory
≈
Number of Keys
× Average Key+Value Size
```

Ví dụ:

```text
10M keys
1KB/key
```

Raw:

```text
~10GB
```

Phải cộng:

- object overhead,
- allocator fragmentation,
- replication,
- metadata,
- headroom.

Có thể phải provision lớn hơn đáng kể.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Key Count | Số key |
| Fragmentation | Memory bị phân mảnh |
| Replication | Bản sao dữ liệu |
| Eviction | Redis xóa key khi thiếu memory |

---

# 18. Cache Hit Ratio và DB Capacity

Traffic:

```text
10,000 RPS
```

Cache Hit:

```text
95%
```

DB traffic:

```text
10,000 × 5%
=
500 RPS
```

Nếu hit ratio giảm còn:

```text
80%
```

DB traffic:

```text
10,000 × 20%
=
2,000 RPS
```

DB load tăng:

```text
4x
```

### Công thức

```text
DB Load after Cache
=
Total Read Traffic × Cache Miss Ratio
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Hit Ratio | Tỷ lệ cache hit |
| Miss Ratio | 1 - Hit Ratio |
| Cache Miss Storm | Nhiều request cùng xuống DB |

---

# 19. Kafka Consumer Sizing

Ví dụ:

```text
Producer = 15,000 msg/s
1 consumer capacity = 1,500 msg/s
```

Công thức:

```text
Consumers Required
=
Producer Rate / Consumer Capacity
```

```text
15,000 / 1,500
=
10 consumers
```

Nhưng:

```text
Active Consumers
<=
Partition Count
```

Nếu cần 10 consumers active:

```text
Partition Count >= 10
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Producer Rate | Message/s được gửi |
| Consumer Capacity | Message/s một consumer xử lý |
| Partition | Đơn vị parallelism Kafka |
| Consumer Group | Nhóm consumer |

---

# 20. Kafka Lag Growth

Công thức:

```text
Lag Growth Rate
=
Producer Rate - Consumer Rate
```

Ví dụ:

```text
Producer = 10,000
Consumer = 8,000
```

```text
Lag Growth
=
2,000 msg/s
```

Sau 5 phút:

```text
2,000 × 300
=
600,000 messages
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Consumer Lag | Message chưa xử lý |
| Lag Growth | Tốc độ backlog tăng |
| Backlog | Message tồn đọng |

---

# 21. Kafka Recovery Capacity

Công thức:

```text
Net Recovery Rate
=
Consumer Capacity - Producer Rate
```

Ví dụ:

```text
Lag = 1,000,000
Producer = 5,000/s
Consumer = 7,000/s
```

```text
Recovery Rate = 2,000/s
```

```text
Recovery Time
=
1,000,000 / 2,000
=
500s
≈ 8.3 phút
```

### Điều kiện

```text
Consumer Capacity > Producer Rate
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Recovery Rate | Tốc độ clear backlog |
| Recovery Time | Thời gian clear lag |
| Backlog Recovery | Khả năng xử lý phần tồn đọng |

---

# 22. Kafka Storage Sizing

Input:

```text
Messages/sec
Average message size
Retention
Replication Factor
```

Ví dụ:

```text
10,000 msg/s
5KB/message
```

Ingress:

```text
~50MB/s
```

Một ngày:

```text
50MB × 86,400
≈ 4.32TB/day
```

Nếu retention:

```text
3 days
```

Raw:

```text
~12.96TB
```

Replication Factor = 3:

```text
~38.88TB
```

Chưa tính overhead/compression.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Retention | Thời gian Kafka giữ message |
| Replication Factor | Số bản sao |
| Ingress | Data đi vào broker |
| Broker Storage | Disk Kafka cần |

---

# 23. Network Sizing

Công thức sơ bộ:

```text
Bandwidth
≈
RPS × Payload Size
```

Ví dụ:

```text
5,000 RPS
Response = 100KB
```

```text
5,000 × 100KB
=
500MB/s
```

Cần tính thêm:

- request payload,
- protocol overhead,
- retry,
- internal calls,
- replication,
- cross-zone traffic.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Bandwidth | Lượng data truyền mỗi giây |
| Payload | Data request/response |
| Ingress | Traffic vào |
| Egress | Traffic ra |
| Cross-zone | Traffic giữa zone |

---

# 24. Downstream Capacity

Nếu:

```text
Service A = 2,000 RPS
```

A gọi B:

```text
3 lần/request
```

B phải chịu:

```text
2,000 × 3
=
6,000 RPS
```

Nếu có:

```text
2 retry
```

Worst-case attempts:

```text
3 attempts
```

Traffic có thể lên:

```text
6,000 × 3
=
18,000 calls/s
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Fan-out | Một request gọi nhiều service |
| Retry Amplification | Retry làm traffic tăng mạnh |
| Downstream Capacity | Capacity của service được gọi |

---

# 25. Queue / Buffer Sizing

Ví dụ:

```text
Producer Burst = 10,000 msg/s
Consumer = 6,000 msg/s
Burst Duration = 60s
```

Backlog phát sinh:

```text
(10,000 - 6,000) × 60
=
240,000 messages
```

Queue phải chịu ít nhất:

```text
240k + headroom
```

### Công thức

```text
Backlog Created
=
(Input Rate - Processing Rate)
× Burst Duration
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Queue Depth | Số item đang chờ |
| Buffer | Vùng chứa tạm |
| Burst Capacity | Khả năng chịu traffic tăng đột ngột |

---

# 26. Tính Headroom

Công thức:

```text
Headroom
=
(Capacity - Peak Load)
/
Peak Load
× 100%
```

Ví dụ:

```text
Capacity = 4,000 TPS
Peak = 3,000 TPS
```

```text
Headroom
=
1,000 / 3,000 × 100%
=
33.3%
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Headroom | Capacity dự phòng |
| Peak Load | Tải peak |
| Capacity Margin | Phần dư capacity |

---

# 27. Failure Capacity / N-1 Sizing

Giả sử:

```text
3 AZ
Total capacity = 6,000 TPS
```

Mỗi AZ khoảng:

```text
2,000 TPS
```

Mất 1 AZ:

```text
Remaining Capacity
=
4,000 TPS
```

Nếu production peak:

```text
5,000 TPS
```

=> hệ thống không đạt N-1 capacity.

SA cần xem:

```text
Normal Capacity
Failure Capacity
Recovery Capacity
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| N-1 | Hệ thống vẫn chạy khi mất một đơn vị |
| Failure Capacity | Capacity khi có failure |
| AZ | Availability Zone |
| Failover | Chuyển workload khi node/component fail |

---

# 28. Autoscaling Sizing

Không chỉ chọn:

```text
CPU threshold = 70%
```

Cần tính:

```text
Detection Delay
+
Scale Decision Delay
+
Pod Startup
+
Warm-up
```

Ví dụ:

```text
Spike duration = 2 phút
Pod ready time = 3 phút
```

=> autoscaling theo CPU có thể phản ứng quá chậm.

Có thể cân nhắc metric:

- RPS,
- CPU,
- queue depth,
- Kafka lag,
- active requests.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| HPA | Horizontal Pod Autoscaler |
| Scale-up Delay | Thời gian tăng pod |
| Warm-up Time | Thời gian pod đạt performance ổn định |
| Queue-based Scaling | Scale dựa trên backlog |

---

# 29. Cost per Capacity

Có thể tính:

```text
Cost per TPS
=
Monthly Infrastructure Cost
/
Sustainable TPS
```

Ví dụ:

```text
Monthly cost = $10,000
Capacity = 5,000 TPS
```

```text
Cost/TPS
=
$2 per sustainable TPS/month
```

Hoặc:

```text
Cost per Million Transactions
```

để so sánh các phương án.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Cost/TPS | Chi phí trên một đơn vị capacity |
| Cost Efficiency | Hiệu quả chi phí |
| Performance/Cost | Trade-off hiệu năng và chi phí |

---

# 30. Sizing theo Forecast 3/6/12 tháng

Nên có bảng:

| Period | Peak TPS | Target Capacity | Pods | DB QPS | Kafka msg/s |
|---|---:|---:|---:|---:|---:|
| Current | 1,000 | 1,300 | 6 | 5,000 | 3,000 |
| 3M | 1,300 | 1,690 | 8 | 6,500 | 3,900 |
| 6M | 1,700 | 2,210 | 10 | 8,500 | 5,100 |
| 12M | 2,500 | 3,250 | 14 | 12,500 | 7,500 |

Sizing không nên chỉ nhìn hiện tại.

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Capacity Forecast | Dự báo capacity |
| Forecast Horizon | Mốc 3/6/12 tháng |
| Growth Projection | Dự báo tăng trưởng |

---

# 31. Bảng Sizing Tổng Hợp

Ví dụ output cuối cùng:

| Layer | Input | Calculation | Proposed |
|---|---|---|---|
| Business | Peak 1,000 TPS | ×1.5 growth ×1.3 margin | 1,950 TPS |
| App | 250 TPS/pod | 1,950 / 250 | 8 pods minimum |
| App Headroom | 30% | 8 × 1.3 | ~11 pods |
| DB | 5 queries/tx | 1,950 × 5 | 9,750 QPS |
| DB Pool | Budget 300 | 300 / 11 | <=27/pod |
| Kafka | 3 msg/tx | 1,950 × 3 | 5,850 msg/s |
| Consumer | 1,500 msg/s | 5,850 / 1,500 | 4 active |
| Kafka Partition | 4 active consumers | partitions >= consumers | >=4 |
| Redis | 95% hit target | DB miss 5% | size by keys + overhead |
| Network | 100KB response | 1,950 × 100KB | ~195MB/s |

---

# 32. Validation bằng Load Test

Sizing chỉ là hypothesis.

Phải validate bằng:

```text
Load Test
Stress Test
Soak Test
Failover Test
```

Ví dụ:

```text
Calculated target = 2,000 TPS
```

Test:

```text
Load:   2,000 TPS
Stress: 2,500 -> 4,000 TPS
Soak:   1,500 TPS / 4h
N-1:    2,000 TPS khi mất 1 node/pod group
```

### Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| Sizing Hypothesis | Estimate cần kiểm chứng |
| Validation | Kiểm chứng bằng test |
| Stress Test | Tìm breaking point |
| Soak Test | Kiểm tra dài hạn |

---

# 33. Nếu sizing sai thì điều chỉnh thế nào?

Nếu:

```text
Expected:
1 pod = 300 TPS

Actual:
1 pod = 180 TPS
```

không nên chỉ tăng pod.

Cần xem:

```text
CPU?
DB?
Thread?
Connection?
Redis?
Kafka?
Downstream?
```

Sizing phải iterate:

```text
Estimate
-> Test
-> Measure
-> Adjust
-> Re-test
```

---

# 34. Decision Tree khi không đạt Target

```text
Target không đạt
      |
      +-- App CPU cao?
      |      -> optimize CPU / scale app
      |
      +-- DB CPU/query cao?
      |      -> optimize DB/index/cache
      |
      +-- Pool pending?
      |      -> reduce hold time/query latency
      |
      +-- Kafka lag?
      |      -> consumer/partition/downstream
      |
      +-- Redis bottleneck?
      |      -> hot key/big key/shard
      |
      +-- Downstream chậm?
             -> timeout/cache/async/circuit breaker
```

---

# 35. Checklist Sizing hoàn chỉnh

## Business

```text
[ ] Transactions/day
[ ] Average TPS
[ ] Peak TPS
[ ] Burst TPS
[ ] Growth
[ ] Safety Margin
[ ] SLO
```

## Application

```text
[ ] TPS/pod
[ ] CPU/TPS
[ ] Memory/pod
[ ] Concurrency
[ ] Thread pool
[ ] Scaling efficiency
```

## Database

```text
[ ] DB QPS
[ ] Connection budget
[ ] Query latency
[ ] CPU
[ ] IOPS
[ ] Storage growth
[ ] Index size
```

## Redis

```text
[ ] Key count
[ ] Value size
[ ] Hit ratio
[ ] Memory
[ ] Replication
```

## Kafka

```text
[ ] Producer msg/s
[ ] Consumer capacity
[ ] Partitions
[ ] Message size
[ ] Retention
[ ] Lag recovery
```

## Network

```text
[ ] RPS
[ ] Payload
[ ] Ingress
[ ] Egress
```

## Availability

```text
[ ] N-1 capacity
[ ] Failover
[ ] Recovery
[ ] AZ/node loss
```

## Cost

```text
[ ] Cost/month
[ ] Cost/TPS
[ ] Cost/transaction
```

---

# 36. Template kết luận sizing

```text
Business Target:
2,000 TPS

SLO:
P95 < 500ms
Error < 0.1%

Calculated Capacity:
2,000 TPS + 30% headroom

Application:
- Sustainable: 250 TPS/pod
- Minimum: 8 pods
- Recommended: 10-11 pods

Database:
- Estimated QPS: 10,000
- Connection budget: 300
- Pool/pod: <= 27-30

Kafka:
- Estimated producer rate: 6,000 msg/s
- Consumer capacity: 1,500 msg/s
- Required active consumers: 4
- Required partitions: >=4

Redis:
- Target hit ratio: >=95%
- Capacity sized by key count + object overhead + replication

Network:
- Estimated outbound: ~200MB/s

Risk:
Database is expected to become the first shared bottleneck.

Validation:
1. Load test at 2,000 TPS
2. Stress to 3,000+ TPS
3. Soak test
4. N-1 failover test
```

---

# 37. Công thức quan trọng cần nhớ

| Bài toán | Công thức |
|---|---|
| Average TPS | `Transactions / Seconds` |
| Peak TPS | `Average TPS × Peak Factor` |
| Required Capacity | `Peak × Growth × Safety Margin` |
| Flow TPS | `Total TPS × Traffic Ratio` |
| Dependency Load | `TPS × Calls per Transaction` |
| Concurrency | `TPS × Response Time` |
| Pods Required | `Target TPS / TPS per Pod` |
| Scaling Efficiency | `Throughput Factor / Resource Factor` |
| DB QPS | `TPS × Queries per Transaction` |
| Pool per Pod | `DB Connection Budget / Pod Count` |
| Cache DB Load | `Traffic × Miss Ratio` |
| Consumers Required | `Producer Rate / Consumer Capacity` |
| Kafka Lag Growth | `Producer Rate - Consumer Rate` |
| Recovery Time | `Lag / (Consumer - Producer)` |
| Queue Backlog | `(Input - Processing) × Duration` |
| Headroom | `(Capacity - Peak) / Peak × 100%` |
| Network Bandwidth | `RPS × Payload Size` |
| Cost/TPS | `Monthly Cost / Sustainable TPS` |

---

# 38. Nguyên tắc cuối cùng

Không sizing theo kiểu:

```text
Traffic x5
=> Pod x5
```

Phải kiểm tra:

```text
Business Load
    ↓
Application
    ↓
Thread / Connection
    ↓
Database
    ↓
Cache
    ↓
Kafka
    ↓
Downstream
    ↓
Network
```

Capacity toàn hệ thống gần đúng sẽ bị giới hạn bởi bottleneck yếu nhất:

```text
System Capacity
≈
min(
App,
DB,
Redis,
Kafka,
Downstream,
Network
)
```

Quy trình đúng:

```text
Estimate
-> Calculate
-> Load Test
-> Find Bottleneck
-> Optimize
-> Re-size
-> Re-test
```

Sizing chỉ đáng tin khi đã được **validate bằng performance test thực tế**.
