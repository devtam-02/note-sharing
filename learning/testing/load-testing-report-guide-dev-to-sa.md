# Hướng dẫn xây dựng báo cáo Load Testing từ Dev đến Solution Architect

> Tài liệu này hướng dẫn cách làm một báo cáo Load Testing theo workflow thực tế: từ xác định mục tiêu, tính workload, thiết kế test, thu thập metric, tìm bottleneck, sizing, tối ưu và re-test. Mỗi bước đều có bảng thuật ngữ ngắn ở cuối để tiện học và tra cứu.

---

# 1. Xác định mục tiêu của bài test

## Mục tiêu

Trước khi chạy tool, cần trả lời:

- Hệ thống cần chịu bao nhiêu TPS/RPS?
- Peak production dự kiến là bao nhiêu?
- Latency chấp nhận được là bao nhiêu?
- Error rate tối đa là bao nhiêu?
- Hệ thống cần chạy ổn định trong bao lâu?
- Có yêu cầu về Kafka lag, DB connection, CPU, memory không?
- Có cần đo khả năng recovery sau spike/failure không?

Không nên đặt mục tiêu mơ hồ như:

```text
Test xem hệ thống chịu được bao nhiêu.
```

Nên đặt tiêu chí cụ thể:

```text
Target load: 2,000 TPS
P95 < 500ms
P99 < 1,000ms
Error rate < 0.1%
Duration: 60 phút
CPU < 75%
Kafka lag không tăng liên tục
Không OOM/restart
```

Ví dụ Acceptance Criteria:

| Tiêu chí | Target |
|---|---:|
| TPS | >= 2,000 |
| P95 | < 500 ms |
| P99 | < 1,000 ms |
| Error Rate | < 0.1% |
| CPU | < 75% |
| Memory | Ổn định |
| Kafka Lag | Không tăng liên tục |
| DB Pool | Không saturation |

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| SLO | Mục tiêu chất lượng dịch vụ, ví dụ P95 < 500ms |
| SLA | Cam kết dịch vụ với business/customer |
| TPS | Transactions Per Second |
| RPS | Requests Per Second |
| P95 | 95% request có latency <= giá trị này |
| Error Rate | Tỷ lệ request lỗi |

---

# 2. Xác định loại Performance Test

## Baseline Test

Lấy số liệu chuẩn để so sánh trước/sau tối ưu.

```text
Ví dụ: 200 TPS trong 15 phút
```

## Load Test

Test tại mức tải production dự kiến.

```text
Ví dụ: 2,000 TPS trong 30 phút
```

Mục tiêu: hệ thống có đạt SLO ở tải kỳ vọng không?

## Stress Test

Tăng tải vượt target để tìm saturation/breaking point.

```text
2,000 -> 3,000 -> 4,000 -> 5,000 TPS
```

## Spike Test

Tăng tải đột ngột để kiểm tra burst traffic.

```text
500 TPS -> 5,000 TPS trong 10 giây
```

## Soak Test

Chạy tải lâu để tìm memory leak, thread leak, connection leak, GC degradation.

```text
1,500 TPS trong 4 giờ
```

## Scalability Test

Tăng resource và đo hiệu quả scale.

```text
2 pods -> 4 pods -> 8 pods
```

## Capacity Test

Tìm maximum sustainable throughput mà vẫn đạt SLO.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Baseline Test | Lấy performance chuẩn |
| Load Test | Test tải kỳ vọng |
| Stress Test | Test vượt capacity |
| Spike Test | Test tải tăng đột ngột |
| Soak Test | Test dài hạn |
| Scalability Test | Test hiệu quả scale |
| Capacity Test | Tìm max TPS vẫn đạt SLO |
| Breaking Point | Điểm hệ thống fail rõ rệt |

---

# 3. Tính workload từ dữ liệu business

## Average TPS

Công thức:

```text
Average TPS = Total Transactions / Total Seconds
```

Ví dụ:

```text
10,000,000 transactions/day
```

Một ngày có:

```text
86,400 seconds
```

Nên:

```text
10,000,000 / 86,400 ≈ 116 TPS
```

## Peak TPS

Không sizing production bằng Average TPS.

```text
Peak TPS = Average TPS × Peak Factor
```

Ví dụ:

```text
Average TPS = 116
Peak Factor = 8
Peak TPS = 116 × 8 ≈ 928 TPS
```

Nếu có production metrics:

```text
Peak Factor = Peak TPS / Average TPS
```

## Growth và Safety Margin

```text
Required Capacity
= Peak TPS × Growth Factor × Safety Margin
```

Ví dụ:

```text
Peak = 1,000 TPS
Growth = 50% => 1.5
Safety Margin = 30% => 1.3
```

```text
Required Capacity
= 1,000 × 1.5 × 1.3
= 1,950 TPS
```

Có thể đặt target khoảng 2,000 TPS.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Average TPS | TPS trung bình |
| Peak TPS | TPS cao nhất trong peak |
| Peak Factor | Peak / Average |
| Growth Factor | Hệ số tăng trưởng |
| Safety Margin | Capacity dự phòng |
| Headroom | Capacity còn dư |
| Required Capacity | Capacity mục tiêu |

---

# 4. Xây dựng Workload Model

Production thường có nhiều loại request với tỷ lệ khác nhau.

Ví dụ:

| API | Tỷ lệ |
|---|---:|
| GET Campaign | 40% |
| Create Campaign | 10% |
| Validate | 20% |
| Cashback | 15% |
| Search | 10% |
| Other | 5% |

Nếu:

```text
Total TPS = 2,000
```

Công thức:

```text
Endpoint TPS = Total TPS × Traffic Ratio
```

Ví dụ:

```text
GET Campaign = 2,000 × 40% = 800 TPS
Create Campaign = 2,000 × 10% = 200 TPS
```

Cần mô phỏng gần production về:

- read/write ratio,
- hot endpoint,
- payload size,
- user flow,
- retry,
- async events.

Lưu ý: business TPS không giống internal RPS.

```text
1 Order
-> 1 Promotion call
-> 1 Inventory call
-> 1 Payment call
```

Nếu Order = 1,000 TPS thì downstream có thể nhận ~3,000 RPS, chưa tính retry.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Workload Model | Mô hình traffic mô phỏng production |
| Workload Mix | Tỷ lệ các loại request |
| Traffic Ratio | % traffic của một API |
| Internal RPS | Request nội bộ giữa service |
| Read/Write Ratio | Tỷ lệ đọc/ghi |
| Hot Endpoint | API có traffic lớn |

---

# 5. Xác định VU, Concurrency và Arrival Rate

## VU không phải TPS

Sai:

```text
100 VU = 100 TPS
```

VU là số user giả lập; TPS là throughput.

## Little's Law

```text
Concurrency ≈ Throughput × Response Time
```

Ví dụ:

```text
TPS = 2,000
Average Latency = 200ms = 0.2s
```

```text
Concurrency ≈ 2,000 × 0.2 = 400
```

Tức trung bình có ~400 request in-flight.

## Nếu có Think Time

Approximation:

```text
VU ≈ TPS × (Response Time + Think Time)
```

Ví dụ:

```text
TPS = 100
Response = 0.5s
Think Time = 2s
VU ≈ 100 × 2.5 = 250
```

## Open vs Closed Model

Closed Model:

```text
Send -> Wait -> Think -> Send
```

Nếu server chậm thì load generator cũng tạo traffic chậm đi.

Open Model:

```text
Arrival Rate = 2,000 request/s
```

Tool cố duy trì arrival rate gần như độc lập với response time.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| VU | Virtual User |
| Concurrency | Số request active đồng thời |
| Arrival Rate | Tốc độ request đến hệ thống |
| Think Time | Thời gian user chờ giữa action |
| Open Model | Traffic theo arrival rate |
| Closed Model | Traffic phụ thuộc VU và response time |
| In-flight Request | Request đang xử lý/chờ xử lý |

---

# 6. Chuẩn bị Test Environment

Báo cáo phải mô tả rõ môi trường.

| Component | Configuration |
|---|---|
| Application | 4 pods |
| CPU/pod | request 500m / limit 2 CPU |
| Memory/pod | request 512Mi / limit 3Gi |
| JVM | Java 21 |
| DB | MariaDB, 8 CPU, 32GB RAM |
| Hikari Pool | 30 connections/pod |
| Redis | 3 nodes |
| Kafka | 6 partitions |
| Consumer | 6 instances |
| Tool | k6 / JMeter / Gatling |

Nếu chỉ ghi:

```text
Max TPS = 3,000
```

mà không ghi environment thì kết quả khó tái sử dụng.

Cần ghi mọi thay đổi giữa các lần test: CPU, memory, pod count, DB size, partitions, config.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Test Environment | Môi trường chạy test |
| Pod | Instance ứng dụng trong Kubernetes |
| Resource Request | Tài nguyên Kubernetes đảm bảo |
| Resource Limit | Giới hạn tài nguyên pod |
| Connection Pool | Pool kết nối DB tái sử dụng |
| Partition | Đơn vị parallelism của Kafka |

---

# 7. Chuẩn bị Test Data

Data phải đủ gần production.

Ví dụ:

```text
Production = 100M rows
Test = 10K rows
```

Kết quả query có thể nhanh giả tạo.

Cần chú ý:

- data volume,
- cardinality,
- distribution,
- hot key,
- payload size,
- index size,
- cache state.

Cardinality ví dụ:

```text
status: 3 giá trị -> low cardinality
customer_id: 100M giá trị -> high cardinality
```

Cardinality ảnh hưởng query plan, index, cache, partitioning.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Cardinality | Số giá trị khác nhau |
| Data Distribution | Cách dữ liệu phân bố |
| Hot Key | Key bị truy cập quá nhiều |
| Dataset Size | Kích thước dữ liệu |
| Payload Size | Kích thước request/response |

---

# 8. Warm-up

Java application cần warm-up do:

- JIT,
- class loading,
- DB connections,
- HTTP connections,
- Redis cache,
- DB buffer/cache.

Ví dụ:

```text
Warm-up = 5 phút
```

Không lấy số liệu warm-up làm kết quả chính.

Dấu hiệu chưa ổn định:

- latency giảm dần,
- CPU dao động mạnh,
- pool chưa ổn định,
- cache hit tăng liên tục.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Warm-up | Làm hệ thống ổn định trước khi đo |
| JIT | Just-In-Time compilation |
| Cache Warm-up | Nạp dữ liệu thường dùng vào cache |
| Steady State | Trạng thái hệ thống tương đối ổn định |

---

# 9. Thiết kế Load Profile

Ví dụ:

| Thời gian | Tải |
|---|---:|
| 0-5 phút | Warm-up |
| 5-15 phút | 500 TPS |
| 15-25 phút | 1,000 TPS |
| 25-35 phút | 1,500 TPS |
| 35-45 phút | 2,000 TPS |
| 45-55 phút | 2,500 TPS |
| 55-60 phút | Recovery |

Mục tiêu là thấy rõ TPS nào bắt đầu saturation.

Sau peak/spike cần xem:

- lag có giảm không,
- CPU có về baseline không,
- memory có release không,
- connection pool có hồi phục không.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Ramp-up | Giai đoạn tăng tải |
| Steady State | Giai đoạn giữ tải ổn định |
| Peak | Mức tải cao dự kiến |
| Spike | Tải tăng đột ngột |
| Recovery | Khả năng hệ thống hồi phục |

---

# 10. Thu thập Client-side Metrics

Các metric tối thiểu:

| Metric | Ý nghĩa |
|---|---|
| Offered TPS | Load tool cố tạo |
| Actual TPS | Throughput thực tế |
| P50 | Median latency |
| P95 | Tail latency |
| P99 | Extreme tail latency |
| Max | Request chậm nhất |
| Success Rate | Tỷ lệ thành công |
| Error Rate | Tỷ lệ lỗi |
| Timeout Rate | Tỷ lệ timeout |

## Throughput

```text
Throughput = Completed Requests / Time
```

Ví dụ:

```text
600,000 / 300s = 2,000 TPS
```

## Error Rate

```text
Error Rate = Failed Requests / Total Requests × 100%
```

## Success Rate

```text
Success Rate = Successful Requests / Total Requests × 100%
```

## Average Latency

```text
Average Latency = Σ Response Time / Number of Requests
```

Nhưng không nên chỉ dùng average; cần P95/P99.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Offered Load | Load muốn gửi |
| Actual Throughput | Load xử lý thực tế |
| P50 | Median |
| P95 | 95th percentile |
| P99 | 99th percentile |
| Timeout Rate | Tỷ lệ timeout |

---

# 11. Thu thập Application/JVM Metrics

Application:

```text
CPU
Memory
Active Requests
Thread Pool
Queue
Rejected Tasks
```

JVM:

```text
Heap Used
Heap After GC
Allocation Rate
GC Count
GC Pause
Thread Count
Metaspace
Native Memory
```

Dấu hiệu memory leak:

```text
Heap after GC:
1.0GB -> 1.2GB -> 1.5GB -> 1.8GB -> 2.1GB
```

Nếu P99 spike cùng GC pause spike thì cần điều tra GC/allocation.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Heap | Memory chứa Java objects |
| GC | Garbage Collection |
| GC Pause | Thời gian pause do GC |
| Allocation Rate | Tốc độ tạo object |
| Metaspace | Memory metadata class |
| Native Memory | Memory JVM ngoài heap |

---

# 12. Thu thập Thread Pool Metrics

Theo dõi:

```text
Active Threads
Max Threads
Queue Size
Rejected Tasks
```

Ví dụ:

```text
Active = 200/200
Queue = 10,000
CPU = 40%
```

Có thể thread pool saturated hoặc thread đang blocking.

Kiểm tra DB wait, HTTP wait, lock, synchronized, Future.get(), blocking I/O.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Thread Pool | Nhóm thread tái sử dụng |
| Active Thread | Thread đang xử lý |
| Queue | Work đang chờ thread |
| Rejected Task | Task bị từ chối |
| Blocking | Thread chờ I/O/lock |

---

# 13. Thu thập Database Metrics

| Metric | Ý nghĩa |
|---|---|
| Query Latency | Query mất bao lâu |
| Slow Query | Query chậm bất thường |
| DB CPU | CPU database |
| Active Connections | Kết nối đang dùng |
| Pending Connections | Request chờ connection |
| Lock Wait | Chờ lock |
| Deadlock | Transaction conflict |
| Rows Examined | Số row DB scan |
| IOPS | Disk operations/sec |

## Pool Utilization

```text
Pool Utilization = Active Connections / Max Pool Size × 100%
```

Ví dụ:

```text
27 / 30 × 100 = 90%
```

90% chưa chắc là lỗi nếu pending = 0. Nếu pending tăng liên tục thì đang saturation.

Ví dụ DB bottleneck:

```text
App CPU = 35%
DB CPU = 96%
API P95 = 1.5s
DB query latency tăng mạnh
Hikari pending tăng
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Query Latency | Thời gian query DB |
| Slow Query | Query chậm bất thường |
| Connection Pool | Pool connection DB |
| Lock Wait | Thời gian chờ lock |
| Deadlock | Transaction chờ nhau vòng tròn |
| Rows Examined | Số row DB kiểm tra |
| IOPS | Input/Output Operations Per Second |

---

# 14. Thu thập Kafka Metrics

Nếu flow dùng Kafka, cần đo:

```text
Producer Rate
Consumer Rate
Consumer Lag
Lag Growth Rate
Max Lag
Recovery Time
Processing Latency
Retry
DLQ
Rebalance
```

## Lag Growth Rate

```text
Lag Growth Rate = Producer Rate - Consumer Rate
```

Ví dụ:

```text
10,000 - 8,000 = 2,000 msg/s
```

Sau 5 phút:

```text
2,000 × 300 = 600,000 messages
```

## Recovery Time

```text
Recovery Time
= Current Lag / (Consumer Capacity - Producer Rate)
```

Ví dụ:

```text
Lag = 1,000,000
Producer = 5,000/s
Consumer = 7,000/s
```

```text
Recovery = 1,000,000 / 2,000 = 500s ≈ 8.3 phút
```

Điều kiện: Consumer Capacity > Producer Rate.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Producer Rate | Message vào Kafka mỗi giây |
| Consumer Rate | Message xử lý mỗi giây |
| Consumer Lag | Message chưa xử lý |
| Lag Growth Rate | Tốc độ lag tăng |
| Recovery Time | Thời gian clear backlog |
| Rebalance | Phân phối lại partition |
| DLQ | Dead Letter Queue |

---

# 15. Thu thập Redis/Cache Metrics

Theo dõi:

```text
Hit Ratio
Latency
CPU
Memory
Eviction
Hot Key
Big Key
Network
```

## Cache Hit Ratio

```text
Hit Ratio = Hits / (Hits + Misses) × 100%
```

Ví dụ:

```text
950 / (950 + 50) = 95%
```

Ảnh hưởng DB:

```text
Traffic = 10,000 RPS
Hit = 95% -> DB ~500 RPS
Hit = 80% -> DB ~2,000 RPS
```

DB load tăng 4 lần dù user traffic không đổi.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Cache Hit | Tìm thấy dữ liệu trong cache |
| Cache Miss | Không tìm thấy dữ liệu |
| Hit Ratio | Tỷ lệ request phục vụ từ cache |
| Eviction | Cache xóa key do policy/memory |
| Hot Key | Key bị truy cập quá nhiều |
| Big Key | Key chứa dữ liệu quá lớn |

---

# 16. Thu thập Downstream và Network Metrics

Downstream:

```text
RPS
P95/P99
Timeout
Retry
Connection Pool
Error Rate
```

## Fan-out

```text
Service A = 2,000 RPS
A gọi B 3 lần/request
```

```text
B traffic = 2,000 × 3 = 6,000 RPS
```

## Retry Amplification

Nếu thêm tối đa 2 retry:

```text
3 attempts/request
Worst-case = 6,000 × 3 = 18,000 calls/s
```

## Network Throughput

```text
Network Throughput ≈ TPS × Payload Size
```

Ví dụ:

```text
5,000 TPS × 100KB ≈ 500MB/s
```

chưa tính protocol overhead và retry.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Downstream | Service được gọi tới |
| Fan-out | Một request tạo nhiều downstream calls |
| Retry Amplification | Retry làm traffic tăng nhiều lần |
| Payload | Dữ liệu request/response |
| Network Throughput | Lượng data qua network mỗi giây |

---

# 17. Đo End-to-End Business Latency

Với async system:

```text
API Latency != Business Latency
```

Ví dụ:

```text
API = 100ms
Kafka Wait = 2s
Consumer = 1s
```

Business latency ~3.1s.

Công thức:

```text
End-to-End Latency
= Final Business State Timestamp - Initial Request Timestamp
```

Rất quan trọng với flow:

```text
API -> Kafka -> Service A -> Kafka -> Service B -> DB
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| API Latency | Thời gian API trả response |
| Business Latency | Thời gian hoàn tất nghiệp vụ |
| End-to-End Latency | Latency toàn flow |
| Async Flow | Flow bất đồng bộ |

---

# 18. Xác định Saturation Point

Bảng mẫu:

| Offered TPS | Actual TPS | P95 | P99 | Error |
|---:|---:|---:|---:|---:|
| 500 | 500 | 100ms | 150ms | 0% |
| 1,000 | 995 | 130ms | 210ms | 0% |
| 1,500 | 1,480 | 190ms | 320ms | 0.02% |
| 2,000 | 1,920 | 420ms | 800ms | 0.1% |
| 2,500 | 2,000 | 1.4s | 3s | 2% |
| 3,000 | 2,050 | 4s | 9s | 8% |

Từ 2,000 -> 3,000 offered TPS, actual TPS tăng rất ít nhưng latency/error tăng mạnh.

=> saturation khoảng 2,000 TPS.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Saturation Point | Điểm throughput không tăng tương xứng |
| Breaking Point | Điểm hệ thống lỗi nghiêm trọng |
| Offered TPS | Tải muốn gửi |
| Actual TPS | Tải xử lý thực tế |
| Tail Latency | P95/P99 latency |

---

# 19. Xác định Sustainable Capacity

Không lấy maximum observed TPS.

```text
Sustainable Capacity = Maximum TPS vẫn đáp ứng SLO
```

Ví dụ SLO:

```text
P95 < 500ms
Error < 0.1%
```

Nếu 2,000 TPS vừa chạm giới hạn này, thì sustainable capacity xấp xỉ 2,000 TPS.

Production peak nên thấp hơn để còn headroom.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Sustainable Capacity | Tải tối đa vẫn đạt SLO |
| Maximum TPS | TPS cao nhất quan sát, chưa chắc dùng được |
| Headroom | Capacity dự phòng |
| Production Peak | Peak thực tế production |

---

# 20. Phân tích Bottleneck bằng USE Method

Với mỗi resource:

```text
U = Utilization
S = Saturation
E = Errors
```

Ví dụ DB Pool:

```text
Utilization = active/max
Saturation = pending requests
Errors = connection timeout
```

Ví dụ Kafka:

```text
Utilization = processing rate
Saturation = lag
Errors = retry/DLQ
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Utilization | Resource đang được dùng bao nhiêu |
| Saturation | Work phải chờ resource |
| Error | Resource bắt đầu gây lỗi |
| USE Method | Utilization-Saturation-Errors |

---

# 21. Nhận diện Bottleneck theo pattern

| Dấu hiệu | Khả năng |
|---|---|
| CPU 100%, DB bình thường | CPU bottleneck |
| CPU thấp, DB CPU cao | DB bottleneck |
| CPU thấp, DB pool full | DB/pool/query bottleneck |
| Thread queue tăng | Blocking/thread saturation |
| P99 tăng cùng GC Pause | JVM/GC pressure |
| Kafka lag tăng đều | Consumer capacity thiếu |
| Scale consumer không cải thiện | Partition/downstream bottleneck |
| Scale pod không tăng TPS | Shared resource bottleneck |
| Cache hit giảm, DB tăng | Cache issue |
| Lock wait cao | Transaction contention |

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Bottleneck | Thành phần giới hạn toàn hệ thống |
| Shared Resource | Resource dùng chung như DB/Redis/Kafka |
| Contention | Nhiều request tranh resource |
| GC Pressure | Áp lực Garbage Collection |
| Thread Saturation | Thread pool hết capacity |

---

# 22. Đưa ra phương án tối ưu

## CPU bottleneck

Kiểm tra:

```text
Algorithm
Serialization
Regex
Crypto
Compression
Logging
Allocation
```

Workflow:

```text
Profile -> Find hot path -> Optimize -> Re-test
```

## DB bottleneck

Kiểm tra:

```text
Slow Query
EXPLAIN
N+1
Index
Rows Scanned
Lock
Transaction
Pagination
```

Tối ưu:

```text
Index
Query rewrite
Batch
Projection
Cache
Read Replica
Partition
```

## Thread bottleneck

Tối ưu:

```text
Timeout
Bulkhead
Pool sizing
Async
Reduce blocking
```

## Kafka bottleneck

Tối ưu:

```text
Consumer logic
Batch
Scale consumer
Increase partitions
Optimize downstream
```

## Redis bottleneck

Tối ưu:

```text
Shard
Reduce value size
TTL strategy
Correct data structure
Local cache
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Profiling | Đo code tốn CPU/memory ở đâu |
| EXPLAIN | Xem query execution plan |
| N+1 | Một query cha kéo theo N query con |
| Bulkhead | Cô lập resource giữa các luồng |
| Batch | Xử lý nhiều item trong một lần |
| Read Replica | DB replica phục vụ đọc |

---

# 23. Re-test sau tối ưu

Phải giữ gần như giống nhau:

```text
Same workload
Same data
Same environment
Same duration
```

Bảng so sánh:

| Metric | Before | After |
|---|---:|---:|
| TPS | 2,000 | 3,000 |
| P95 | 800ms | 350ms |
| P99 | 2s | 700ms |
| Error | 1% | 0.05% |
| DB CPU | 98% | 65% |

Nếu workload khác thì khó chứng minh optimization.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Re-test | Chạy lại sau tối ưu |
| Baseline | Mốc so sánh ban đầu |
| Regression | Hiệu năng giảm sau thay đổi |
| Controlled Comparison | So sánh trong điều kiện giống nhau |

---

# 24. Stress Test sau tối ưu

Sau khi đạt target Load Test:

```text
Target = 2,000 TPS
```

Stress tiếp:

```text
2,500 -> 3,000 -> 4,000 TPS
```

Mục tiêu:

- saturation point mới,
- breaking point mới,
- failure behavior,
- recovery behavior.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| New Saturation Point | Điểm nghẽn mới sau tối ưu |
| Failure Behavior | Cách hệ thống fail |
| Graceful Degradation | Giảm chất lượng có kiểm soát |
| Recovery | Khả năng hồi phục |

---

# 25. Soak Test

Ví dụ:

```text
Load = 70-80% expected peak
Duration = 4-8 giờ
```

Theo dõi:

```text
Heap trend
GC
Thread count
DB connections
Kafka lag
Redis memory
Error trend
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Soak Test | Test dài hạn |
| Memory Leak | Memory tăng và không giải phóng |
| Connection Leak | Connection không trả pool |
| Performance Degradation | Performance giảm theo thời gian |

---

# 26. Tính số Pod sơ bộ

Nếu benchmark:

```text
1 pod sustainable = 350 TPS
```

Target:

```text
3,000 TPS
```

Công thức:

```text
Pods Required = Target TPS / Sustainable TPS per Pod
```

```text
3,000 / 350 ≈ 8.57
```

=> ít nhất 9 pods.

Nếu thêm 30% headroom:

```text
9 × 1.3 ≈ 12 pods
```

Phải re-test vì scaling không luôn tuyến tính.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| TPS per Pod | Throughput một pod chịu được |
| Pod Sizing | Tính số pod cần thiết |
| Headroom | Tài nguyên dự phòng |
| Horizontal Scaling | Tăng số instance/pod |

---

# 27. Tính Scaling Efficiency

Ví dụ:

```text
2 pods = 1,000 TPS
4 pods = 1,800 TPS
```

```text
Resource Factor = 4/2 = 2
Throughput Factor = 1,800/1,000 = 1.8
```

Công thức:

```text
Scaling Efficiency
= Throughput Factor / Resource Factor × 100%
```

```text
1.8 / 2 × 100% = 90%
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Scaling Efficiency | Hiệu quả tăng throughput khi tăng resource |
| Resource Factor | Hệ số tăng tài nguyên |
| Throughput Factor | Hệ số tăng throughput |
| Linear Scaling | Resource x2, throughput gần x2 |

---

# 28. Tính Headroom

Công thức:

```text
Headroom
= (Capacity - Peak Load) / Peak Load × 100%
```

Ví dụ:

```text
Capacity = 4,000 TPS
Peak = 3,000 TPS
```

```text
Headroom = 33.3%
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Capacity | Mức tải hệ thống chịu được |
| Peak Load | Peak production dự kiến |
| Headroom | Capacity còn dư |
| Safety Margin | Phần dự phòng dùng khi sizing |

---

# 29. Tính DB Connection Pool sơ bộ

Ví dụ:

```text
DB safe connection budget = 300
Pods = 10
```

Approximation:

```text
Pool per pod <= 300 / 10 = 30
```

Nhưng phải chừa cho:

- service khác,
- migration,
- admin,
- monitoring.

Không tăng pool nếu DB đã saturation.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| DB Connection Budget | Tổng connection DB dành cho service |
| Pool per Pod | Connection pool mỗi pod |
| Pending Connection | Request chờ connection |
| DB Saturation | DB không còn capacity |

---

# 30. Tính Kafka Consumer sơ bộ

Ví dụ:

```text
Producer = 15,000 msg/s
1 consumer = 1,500 msg/s
```

Công thức:

```text
Consumers Required = Producer Rate / Consumer Capacity
```

```text
15,000 / 1,500 = 10 consumers
```

Cần:

```text
Partition Count >= 10
```

nếu muốn 10 consumer active trong cùng group.

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Consumer Capacity | Message/s một consumer xử lý được |
| Consumer Group | Nhóm consumer chia partitions |
| Partition Count | Số partition của topic |
| Parallelism | Mức xử lý song song |

---

# 31. Báo cáo phải có Bottleneck Evidence

Không nên viết:

```text
DB có vẻ chậm.
```

Nên viết:

```text
At 2,500 TPS:
- App CPU = 52%
- DB CPU = 96%
- Query P95: 40ms -> 380ms
- Hikari pending: 0 -> 120
- API P95: 350ms -> 1.5s

Conclusion:
Database is the primary bottleneck.
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Evidence | Dữ liệu chứng minh kết luận |
| Correlation | Metric thay đổi cùng thời điểm |
| Root Cause | Nguyên nhân gốc |
| Primary Bottleneck | Bottleneck chính |
| Secondary Bottleneck | Bottleneck tiếp theo có thể xuất hiện |

---

# 32. Bảng kết quả tổng hợp nên có

| Load | Actual TPS | P50 | P95 | P99 | Error | App CPU | DB CPU | DB Pool | Kafka Lag |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 500 | 500 | 80ms | 120ms | 180ms | 0% | 20% | 25% | 30% | 0 |
| 1,000 | 995 | 90ms | 160ms | 250ms | 0% | 35% | 45% | 45% | 0 |
| 1,500 | 1,480 | 110ms | 230ms | 400ms | 0.02% | 50% | 65% | 60% | 500 |
| 2,000 | 1,930 | 180ms | 480ms | 900ms | 0.1% | 62% | 85% | 85% | 5k |
| 2,500 | 2,050 | 500ms | 1.5s | 4s | 2% | 65% | 98% | 100% | 120k |

Có thể đọc nhanh:

```text
Saturation khoảng 2,000 TPS
DB bắt đầu bottleneck
Kafka lag tăng sau 2,000 TPS
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Result Matrix | Bảng kết quả theo từng mức tải |
| Correlation Analysis | So sánh metric giữa các layer |
| Load Step | Một mức tải trong test |
| Saturation Threshold | Ngưỡng bắt đầu saturation |

---

# 33. Cấu trúc báo cáo Load Testing hoàn chỉnh

Một báo cáo nên có:

```text
1. Executive Summary
2. Test Objective
3. SLO / Acceptance Criteria
4. System Architecture
5. Test Environment
6. Test Data
7. Workload Model
8. Test Scenarios
9. Monitoring Metrics
10. Test Results
11. Saturation Analysis
12. Bottleneck Analysis
13. Capacity Analysis
14. Sizing Recommendation
15. Optimization Recommendation
16. Risks / Limitations
17. Conclusion
18. Re-test Plan
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Executive Summary | Tóm tắt cho người ra quyết định |
| Acceptance Criteria | Điều kiện pass/fail |
| Capacity Analysis | Phân tích tải bền vững |
| Sizing Recommendation | Đề xuất tài nguyên |
| Risk | Rủi ro còn tồn tại |
| Limitation | Giới hạn của bài test |

---

# 34. Template Executive Summary

```text
Objective:
Validate system capacity at 2,000 TPS.

Result:
System satisfies SLO up to approximately 1,900-2,000 TPS.

Observed:
- P95 at 2,000 TPS: 480ms
- P99: 900ms
- Error: 0.1%
- App CPU: 62%
- DB CPU: 85%
- Kafka lag: 5k and recoverable

At 2,500 TPS:
- Actual TPS only 2,050
- P95 increased to 1.5s
- Error increased to 2%
- DB CPU reached 98%
- DB pool reached 100%

Conclusion:
Primary bottleneck is database capacity.

Recommendation:
1. Optimize slow queries and indexes.
2. Review N+1.
3. Reduce transaction duration.
4. Consider cache for read-heavy queries.
5. Re-test same workload.
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Executive Summary | Kết luận ngắn gọn của toàn báo cáo |
| Observed | Số liệu thực tế quan sát |
| Recommendation | Đề xuất cải thiện |
| Re-test Plan | Kế hoạch chạy lại sau tối ưu |

---

# 35. Công thức quan trọng nên nhớ

| Bài toán | Công thức |
|---|---|
| Average TPS | `Transactions / Seconds` |
| Peak TPS | `Average TPS × Peak Factor` |
| Required Capacity | `Peak × Growth × Safety Margin` |
| Endpoint TPS | `Total TPS × Traffic Ratio` |
| Concurrency | `TPS × Avg Response Time` |
| VU approximation | `TPS × (Response + Think Time)` |
| Error Rate | `Errors / Total × 100%` |
| Success Rate | `Success / Total × 100%` |
| Pool Utilization | `Active / Max × 100%` |
| Cache Hit Ratio | `Hits / (Hits + Misses)` |
| Kafka Lag Growth | `Producer - Consumer` |
| Kafka Recovery Time | `Lag / (Consumer - Producer)` |
| Pods Required | `Target TPS / TPS per Pod` |
| Headroom | `(Capacity - Peak) / Peak × 100%` |
| Scaling Efficiency | `Throughput Factor / Resource Factor × 100%` |
| Network Throughput | `TPS × Payload Size` |

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Formula Sheet | Bảng công thức tra nhanh |
| Approximation | Công thức ước lượng, không phải tuyệt đối |
| Sustainable TPS | TPS bền vững dưới SLO |
| Capacity Planning | Lập kế hoạch capacity từ workload |

---

# 36. Góc nhìn Dev và Solution Architect

## Dev cần trả lời

```text
Code nào chậm?
Query nào chậm?
Thread nào block?
GC có vấn đề không?
Pool nào đầy?
Kafka consumer chậm ở đâu?
```

## SA cần trả lời thêm

```text
Business cần bao nhiêu TPS?
Current sustainable capacity là bao nhiêu?
Bottleneck toàn hệ thống nằm ở đâu?
Nếu traffic x5 thì component nào fail trước?
Scale app có đủ không?
DB chịu được không?
Kafka cần bao nhiêu partitions?
Cache fail thì DB có chịu nổi không?
Headroom bao nhiêu?
Failure capacity bao nhiêu?
Cost để đạt target là bao nhiêu?
```

Tư duy cần chuyển từ:

```text
Optimize one service
```

sang:

```text
Business Load
    ↓
Application
    ↓
Database
    ↓
Cache
    ↓
Kafka
    ↓
Downstream
    ↓
Infrastructure
```

Mục tiêu cuối cùng:

```text
Đảm bảo toàn bộ business flow:
- đủ throughput,
- đạt latency SLO,
- còn headroom,
- recover được,
- scale được,
- chi phí hợp lý.
```

### Thuật ngữ cần nhớ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| Dev View | Tập trung implementation/component |
| SA View | Tập trung end-to-end capacity và architecture |
| Failure Capacity | Capacity khi một thành phần bị mất |
| Cost Efficiency | Hiệu năng đạt được trên chi phí bỏ ra |
| End-to-End | Toàn bộ business flow |
