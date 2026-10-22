# Đánh giá Chỉ số Latency trong Hệ thống Phân tán

## Mục lục

1. [Tổng quan: Latency là gì](#1-tổng-quan-latency-là-gì)
2. [Thành phần cấu tạo Latency](#2-thành-phần-cấu-tạo-latency)
3. [Tại sao Average là thước đo sai lầm](#3-tại-sao-average-là-thước-đo-sai-lầm)
4. [Percentile — Thước đo đúng](#4-percentile--thước-đo-đúng)
5. [Phân phối xác suất của Latency](#5-phân-phối-xác-suất-của-latency)
6. [P95 vs P99: Khác biệt ý nghĩa giám sát](#6-p95-vs-p99-khác-biệt-ý-nghĩa-giám-sát)
7. [Hiệu ứng Fan-out Amplification](#7-hiệu-ứng-fan-out-amplification)
8. [Little's Law và Queueing Theory](#8-littles-law-và-queueing-theory)
9. [Chiến lược Alerting theo Percentile](#9-chiến-lược-alerting-theo-percentile)
10. [Các sai lầm phổ biến khi đo Latency](#10-các-sai-lầm-phổ-biến-khi-đo-latency)
11. [Công cụ và Cấu trúc dữ liệu](#11-công-cụ-và-cấu-trúc-dữ-liệu)
12. [Checklist thực hành](#12-checklist-thực-hành)

---

## 1. Tổng quan: Latency là gì

**Latency** (Độ trễ) là tổng thời gian cần thiết để một yêu cầu hoàn tất chu trình xử lý, tính từ lúc client gửi request đến khi nhận được response cuối cùng.

- **Đơn vị**: milliseconds (ms)
- **Mục tiêu**: càng thấp càng tốt
- **Bản chất**: là góc nhìn **vi mô** — đo trải nghiệm của **từng request đơn lẻ**

Latency khác biệt cơ bản với Throughput (thông lượng — số requests/giây hệ thống xử lý được). Hai đại lượng này **không độc lập** mà ràng buộc chặt chẽ qua lý thuyết hàng đợi.

---

## 2. Thành phần cấu tạo Latency

### 2.1. Tại một nút xử lý

```
R = Wq + S
```

| Ký hiệu | Tên | Ý nghĩa |
|----------|-----|---------|
| **R** | Response Time / Residence Time | Tổng latency của request |
| **Wq** | Waiting Time | Thời gian chờ trong hàng đợi để được cấp phát tài nguyên |
| **S** | Service Time | Thời gian hệ thống thực sự xử lý tác vụ |

### 2.2. Trên đường truyền mạng

```
Network Latency = Propagation + Transmission + Queuing + Processing
```

| Thành phần | Công thức | Phụ thuộc vào |
|------------|-----------|---------------|
| **Propagation** | distance ÷ speed of medium | Khoảng cách địa lý, tốc độ ánh sáng trong fiber |
| **Transmission** | message size ÷ bandwidth | Kích thước gói tin, băng thông kênh |
| **Queuing** | biến thiên | Tình trạng tải tại router/switch trung gian |
| **Processing** | biến thiên | Thời gian xử lý header, routing tại mỗi hop |

---

## 3. Tại sao Average là thước đo sai lầm

### 3.1. Ví dụ minh họa

Xét 2 service, mỗi service xử lý 10 requests:

**Service A** (ổn định):
```
48, 49, 50, 50, 51, 51, 52, 52, 53, 54
```

**Service B** (có long-tail):
```
10, 11, 11, 12, 12, 12, 13, 14, 15, 390
```

| Metric | Service A | Service B |
|--------|:---------:|:---------:|
| **Average** | **51 ms** | **50 ms** |
| **P50** (median) | 51 ms | 12 ms |
| **P95** | 54 ms | **390 ms** |
| **P99** | 54 ms | **390 ms** |

**Kết luận**: Average gần giống nhau (51 vs 50) nhưng P95 chênh nhau **7 lần** (54 vs 390). Service B có vấn đề nghiêm trọng mà average không phát hiện được.

### 3.2. Tại sao Average thất bại

- Average bị **kéo lệch** bởi outlier (giá trị bất thường)
- Average **xóa mất hình dạng** phân phối — không phân biệt được phân phối đều vs phân phối lệch
- Average đưa ra **cùng một điểm số** cho hệ thống khỏe mạnh và hệ thống có vấn đề

> **Phép so sánh**: Average giống như nói "nhiệt độ trung bình cơ thể bạn là 37°C" khi thực tế tay 35°C nhưng trán 40°C — trung bình "bình thường" nhưng bạn đang sốt.

---

## 4. Percentile — Thước đo đúng

### 4.1. Định nghĩa

Sắp xếp tất cả giá trị latency từ thấp đến cao. **P(N)** là giá trị mà **N%** requests có latency thấp hơn hoặc bằng.

```
Ví dụ: 100 requests, sắp xếp tăng dần:

  Request #1   → 5ms    ┐
  Request #2   → 6ms    │
  ...                    ├── 50% nhanh nhất
  Request #50  → 12ms   ┘  ← P50 = 12ms
  ...
  Request #95  → 85ms       ← P95 = 85ms  (chỉ 5% chậm hơn)
  ...
  Request #99  → 340ms      ← P99 = 340ms (chỉ 1% chậm hơn)
  Request #100 → 920ms      ← Max
```

### 4.2. Ý nghĩa từng Percentile

| Metric | Ý nghĩa | Vai trò giám sát | Ai bị ảnh hưởng |
|--------|---------|-------------------|------------------|
| **P50** | Median — nửa requests nhanh hơn giá trị này | Trải nghiệm **đa số** user | Người dùng "trung bình" |
| **P90** | 10% requests chậm hơn | Phát hiện vấn đề **vừa phổ biến** | 1 trong 10 users |
| **P95** | 5% requests chậm hơn | **Cảnh báo chính** — vấn đề đã lan rộng | 1 trong 20 users |
| **P99** | 1% requests chậm hơn | **Phát hiện sớm** — worst-case đang xấu đi | 1 trong 100 users |
| **P99.9** | 0.1% requests chậm hơn | Extreme tail — dùng cho hệ thống critical | 1 trong 1,000 users |

### 4.3. Tần suất thực tế

Với 1 triệu requests/ngày:

| Percentile | Số requests vượt ngưỡng | Tương đương |
|:----------:|:----------------------:|:-----------:|
| P95 | 50,000 / ngày | ~35 / phút |
| P99 | 10,000 / ngày | ~7 / phút |
| P99.9 | 1,000 / ngày | ~0.7 / phút |

→ Ngay cả P99.9 cũng ảnh hưởng **hàng nghìn user mỗi ngày** ở quy mô lớn.

---

## 5. Phân phối xác suất của Latency

### 5.1. Phân phối lý thuyết vs thực tế

Trong Queueing Theory:
- **Mô hình M/M/1**: Thời gian phục vụ tuân theo phân phối **mũ (Exponential)** — đã tạo ra long tail
- **Mô hình M/G/1**: Thời gian phục vụ tuân theo phân phối **tổng quát (General)** — phản ánh thực tế hơn

Trong thực tế, latency thường tuân theo phân phối **Log-normal** hoặc **Multi-modal**:
- **Đỉnh chính**: phần lớn requests nhanh (cache hit, query đơn giản)
- **Đuôi dài (long tail)**: một tỷ lệ nhỏ requests chậm bất thường

### 5.2. Nguyên nhân tạo ra Long Tail

| Nguyên nhân | Ảnh hưởng đến Latency | Tần suất |
|-------------|----------------------|----------|
| **Garbage Collection** (GC pause) | +50ms đến +500ms | Mỗi ~100–1,000 requests |
| **Cache miss** → truy vấn database | +10ms đến +200ms | 1–10% requests |
| **TCP retransmission** | +200ms đến +3,000ms | 0.1–1% packets |
| **Context switch / CPU throttle** | +5ms đến +50ms | Khi tải cao |
| **Lock contention** | +10ms đến +500ms | Khi concurrency cao |
| **Cold start** (serverless/container) | +500ms đến +5,000ms | Request đầu tiên sau idle |
| **Cross-region call** | +60ms đến +200ms | Khi routing thay đổi |

### 5.3. Hệ số biến thiên (Coefficient of Variation)

```
C = σ / μ
```

- **C < 1**: Biến thiên thấp — latency đồng đều
- **C = 1**: Biến thiên chuẩn (phân phối Exponential trong M/M/1)
- **C > 1**: Biến thiên cao — latency không đồng đều, long tail nghiêm trọng

Theo mô hình M/G/1, chỉ số hiệu năng:
```
UPI = C² × ρ / (1 − ρ)
```

Khi C cao, hiệu ứng "hockey-stick" (latency tăng vọt) xuất hiện **sớm hơn nhiều**. Ví dụ: C = 2, ρ = 50% → UPI = 4 — latency đã gấp 4 lần service time ngay ở mức tải 50%.

---

## 6. P95 vs P99: Khác biệt ý nghĩa giám sát

### 6.1. Bảng so sánh cốt lõi

| Tiêu chí | P95 | P99 |
|----------|-----|-----|
| **Nghĩa** | 5% requests chậm hơn giá trị này | 1% requests chậm hơn giá trị này |
| **Tần suất gặp** | 1 trong 20 requests | 1 trong 100 requests |
| **Phản ánh** | Vấn đề **thường xuyên**, ảnh hưởng diện rộng | Vấn đề **hiếm nhưng nghiêm trọng** |
| **Vai trò** | 🔔 **Fire alarm** — xác nhận vấn đề đã lan rộng | 🚨 **Smoke detector** — phát hiện sớm |
| **Độ nhạy** | Ổn định, ít nhiễu | Nhạy hơn, dao động mạnh hơn |
| **Alert** | Alert chính, yêu cầu hành động ngay | Alert cảnh báo, yêu cầu điều tra |

### 6.2. Kịch bản so sánh khả năng phát hiện

**Kịch bản 1 — GC pause** (ảnh hưởng 1% requests):
```
99 requests  → 15ms
1 request    → 500ms (bị GC pause)

→ P95 = 15ms  ❌ Không phát hiện (dưới ngưỡng 5%)
→ P99 = 500ms ✅ Phát hiện GC spike
```

**Kịch bản 2 — DB connection pool cạn** (ảnh hưởng 10% requests):
```
90 requests  → 20ms
10 requests  → 300ms (chờ connection)

→ P95 = 300ms ✅ Phát hiện (trên ngưỡng 5%)
→ P99 = 300ms ✅ Cũng phát hiện
```

**Kịch bản 3 — Suy thoái dần** (memory leak):
```
Giờ 1:  P95 = 80ms    P99 = 150ms   ← Cả hai bình thường
Giờ 2:  P95 = 85ms    P99 = 180ms   ← P99 bắt đầu tăng
Giờ 3:  P95 = 95ms    P99 = 250ms   ← P99 tăng rõ, P95 vẫn "chấp nhận được"
Giờ 4:  P95 = 130ms   P99 = 500ms   ← P95 bắt đầu tăng
Giờ 5:  P95 = 300ms   P99 = 2000ms  ← Cả hai báo động
```

→ **P99 phát hiện sớm hơn 1–2 giờ** so với P95. Đó là thời gian vàng để xử lý.

### 6.3. Quy tắc sử dụng

| Câu hỏi cần trả lời | Dùng metric |
|----------------------|-------------|
| "Đa số user có trải nghiệm tốt không?" | **P50** |
| "Có vấn đề ảnh hưởng diện rộng không?" | **P95** |
| "Có user nào bị trải nghiệm cực tệ không?" | **P99** |
| "Hệ thống có đang suy thoái dần không?" | **P99** (nhạy hơn, phát hiện sớm) |
| "Vấn đề đã lan rộng chưa?" | **P95** (khi P95 tăng = đã phổ biến) |
| "SLA có bị vi phạm không?" | **Tùy SLA** — thường P95 hoặc P99 |

---

## 7. Hiệu ứng Fan-out Amplification

### 7.1. Vấn đề

Trong kiến trúc microservices, một request từ user thường gọi **nhiều services nối tiếp hoặc song song**. Tổng latency bị chi phối bởi **service chậm nhất** (khi gọi song song) hoặc **tổng tất cả** (khi gọi nối tiếp).

### 7.2. Xác suất bị chậm

Xác suất tất cả N services đều nhanh (dưới P99):

```
P(all fast) = 0.99^N
```

| Số services (N) | P(all dưới P99) | % requests bị ≥1 slow call | Với 1M req/ngày |
|:---------------:|:---------------:|:---------------------------:|:---------------:|
| 1 | 99.0% | **1.0%** | 10,000 slow |
| 3 | 97.0% | **3.0%** | 30,000 slow |
| 5 | 95.1% | **4.9%** | 49,000 slow |
| 10 | 90.4% | **9.6%** | 96,000 slow |
| 20 | 81.8% | **18.2%** | 182,000 slow |
| 50 | 60.5% | **39.5%** | 395,000 slow |

### 7.3. Bài học

- Với 20 services (phổ biến ở Google, Amazon): **gần 1 trong 5 requests** bị chậm
- P99 của **toàn hệ thống** luôn tệ hơn P99 của **bất kỳ service đơn lẻ nào**
- Đây là lý do hệ thống quy mô lớn tập trung rất nhiều vào **tail latency optimization**

---

## 8. Little's Law và Queueing Theory

### 8.1. Little's Law

```
L = λ × W
```

| Ký hiệu | Ý nghĩa |
|----------|---------|
| **L** | Số requests đang tồn tại đồng thời trong hệ thống (in-flight) |
| **λ** | Throughput — tốc độ requests/giây |
| **W** | Latency trung bình — thời gian lưu trú của 1 request |

**Tính phổ quát**: Luôn đúng bất kể phân phối xác suất, thứ tự hàng đợi, hay loại tài nguyên (CPU, DB pool, network, disk I/O, message queue).

**Ví dụ**: 1,000 rps × 200ms = **200 concurrent requests** → cần ít nhất 200 worker slots.

### 8.2. Utilization Law

```
ρ = λ × S
```

- **ρ** (rho): Mức sử dụng tài nguyên (0 đến 1)
- **S**: Service time (thời gian xử lý thuần túy, không bao gồm chờ đợi)

### 8.3. Latency theo mô hình M/M/1

```
R = S / (1 − ρ)
```

Bảng minh họa (S = 10ms):

| Utilization (ρ) | Queue Wait | Total R | Hệ số tăng | Đánh giá |
|:---------------:|:----------:|:-------:|:-----------:|:--------:|
| 50% | 10ms | **20ms** | 2× | ✅ An toàn |
| 70% | 23ms | **33ms** | 3.3× | ✅ OK |
| 80% | 40ms | **50ms** | 5× | ⚠️ Cảnh báo |
| 90% | 90ms | **100ms** | 10× | ⚠️ Cao |
| 95% | 190ms | **200ms** | 20× | 🔴 Nguy hiểm |
| 99% | 990ms | **1,000ms** | 100× | 🔴 Tới hạn |

**Quy luật**: Latency tăng **phi tuyến tính** (hyperbolic) khi utilization tiến gần 100%. Không bao giờ chạy production trên 70–80% utilization.

### 8.4. Liên hệ giữa Queueing Theory và Percentile

- Khi **ρ tăng** → hàng đợi dài hơn → **P99 tăng trước**, sau đó P95 tăng theo
- Khi **C (biến thiên) cao** → phân phối latency có đuôi dài hơn → **khoảng cách P50–P99 lớn hơn**
- Giám sát **ρ** kết hợp với **P99** là cách phát hiện sớm nhất: ρ tăng là nguyên nhân, P99 tăng là triệu chứng

---

## 9. Chiến lược Alerting theo Percentile

### 9.1. Ba cấp độ cảnh báo

```
Cấp 1:  P99 > threshold   →  ⚠️  Warning
         "Điều tra — có thể là GC, timeout đơn lẻ, hoặc dấu hiệu sớm"

Cấp 2:  P95 > threshold   →  🚨  Critical
         "Hành động ngay — 5% user bị ảnh hưởng, vấn đề đã lan rộng"

Cấp 3:  P95 VÀ P99 cùng   →  🔥  Incident
         tăng mạnh              "Escalate — hệ thống suy thoái toàn diện"
```

### 9.2. Ngưỡng tham khảo (Web API)

| Metric | SLO Target | Warning | Critical |
|--------|:----------:|:-------:|:--------:|
| **P50** | < 50ms | > 80ms | > 150ms |
| **P95** | < 200ms | > 350ms | > 500ms |
| **P99** | < 500ms | > 800ms | > 1,500ms |

### 9.3. Lưu ý khi đặt alert

- **P99 alert** nên có **thời gian đánh giá dài hơn** (ví dụ: trung bình P99 trong 5 phút) vì P99 dao động mạnh
- **P95 alert** có thể đánh giá trên **cửa sổ ngắn hơn** (1–2 phút) vì ổn định hơn
- Kết hợp alert trên **error rate** + **P99** để phân biệt lỗi vs chậm
- Sử dụng **P50 as baseline** để phát hiện drift dài hạn (degradation chậm theo tuần/tháng)

---

## 10. Các sai lầm phổ biến khi đo Latency

### 10.1. Trung bình hóa percentile giữa các instance

**Sai**: Server A có P99 = 80ms, Server B có P99 = 120ms, Server C có P99 = 300ms → "P99 trung bình = 167ms"

**Đúng**: Gộp toàn bộ raw data từ 3 servers, **sau đó** tính P99 từ tập dữ liệu gộp. Giá trị thực có thể cao hơn 167ms rất nhiều.

> **Quy tắc**: Không bao giờ average percentiles. Merge trước, tính sau.

### 10.2. Coordinated Omission (Bỏ sót phối hợp)

Hầu hết load test tools gửi request tiếp theo **chỉ sau khi** request trước hoàn thành. Khi server chậm, tool gửi ít requests hơn → giai đoạn chậm bị **đại diện không đầy đủ** trong dữ liệu.

**Hệ quả**: P99 đo được **thấp hơn thực tế**.

**Giải pháp**: Dùng tool gửi requests ở **tốc độ cố định** bất kể response:
- `wrk2` (constant-rate mode)
- `Gatling` (open-model)
- `k6` (arrival-rate executor)

### 10.3. Không đủ sample

| Percentile | Số sample tối thiểu để có 1 data point trong tail | Khuyến nghị |
|:----------:|:---------------------------------------------------:|:-----------:|
| P95 | 20 | 500+ |
| P99 | 100 | 1,000+ |
| P99.9 | 1,000 | 10,000+ |

Với mẫu quá nhỏ, percentile bị **nhiễu** (noise) rất mạnh và không đáng tin cậy.

### 10.4. Đo server-side thay vì client-side

```
|←────────────── Client-measured latency ──────────────→|
|                                                        |
| Network → | ←── Server-measured latency ──→ | ← Network |
|           |     (processing only)           |           |
```

Server-side latency **không bao gồm** thời gian mạng và thời gian chờ trong queue phía client. Luôn ưu tiên đo từ **phía client** khi có thể.

### 10.5. Bỏ qua latency khi lỗi

Requests bị timeout hoặc trả lỗi 5xx thường có latency **rất cao** nhưng bị loại khỏi metric vì "không thành công". Điều này khiến percentile trông tốt hơn thực tế.

**Giải pháp**: Bao gồm cả failed requests trong tính toán latency, hoặc tạo metric riêng cho "latency including errors".

---

## 11. Công cụ và Cấu trúc dữ liệu

### 11.1. Cấu trúc dữ liệu cho Percentile Streaming

| Phương pháp | Độ chính xác | Bộ nhớ | Phù hợp |
|-------------|:------------:|:------:|----------|
| **Sorted array** | Chính xác 100% | O(n) — cao | Offline analysis, dataset nhỏ |
| **t-digest** | ~0.1% error ở tail | O(1) — cố định | ★ Streaming, distributed systems |
| **HDR Histogram** | Configurable precision | O(1) — cố định | ★ Low-latency apps, JVM |
| **DDSketch** | Relative error guarantee | O(1) — cố định | Datadog, monitoring platforms |

### 11.2. Công cụ thu thập Metrics

| Công cụ | Percentile support | Ghi chú |
|---------|-------------------|---------|
| **Prometheus** | `histogram_quantile()` | Cần cấu hình bucket phù hợp |
| **Datadog** | Distribution metrics | Tính percentile phía server, chính xác hơn histogram |
| **OpenTelemetry** | Histogram instrument | Chuẩn mở, vendor-neutral |
| **Micrometer** (JVM) | Timer + percentile histogram | Tích hợp tốt với Spring Boot |

### 11.3. Công cụ Load Test

| Công cụ | Tránh Coordinated Omission | Ngôn ngữ |
|---------|:--------------------------:|:--------:|
| **wrk2** | ✅ Có (constant-rate) | C |
| **Gatling** | ✅ Có (open-model) | Scala |
| **k6** | ✅ Có (arrival-rate) | Go/JS |
| **Locust** | ❌ Cần cấu hình thêm | Python |
| **JMeter** | ❌ Mặc định closed-model | Java |

---

## 12. Checklist thực hành

### Khi thiết kế hệ thống

- [ ] Xác định SLO cho P50, P95, P99 (không chỉ average)
- [ ] Tính concurrency cần thiết bằng Little's Law: `L = λ × W`
- [ ] Giữ utilization ρ < 70% cho từng component
- [ ] Tách workload biến thiên cao vào queue riêng (giảm C)
- [ ] Đặt timeout cho mọi external call

### Khi giám sát

- [ ] Thu thập P50, P95, P99 (không chỉ average)
- [ ] Đo từ phía **client**, không chỉ server
- [ ] Bao gồm cả failed requests trong metric latency
- [ ] Dùng t-digest hoặc HDR Histogram cho streaming percentile
- [ ] Không average percentiles giữa các instances — merge raw data

### Khi alert

- [ ] P99 > ngưỡng → Warning (điều tra)
- [ ] P95 > ngưỡng → Critical (hành động ngay)
- [ ] P95 + P99 cùng tăng → Incident (escalate)
- [ ] Giám sát in-flight requests (L) — chỉ báo sớm nhất của overload
- [ ] Auto-scale dựa trên ρ, không đợi timeout

### Khi load test

- [ ] Dùng tool tránh coordinated omission (wrk2, k6 arrival-rate)
- [ ] Đủ sample cho percentile cần đo (≥1,000 cho P99)
- [ ] Test ở nhiều mức tải: 50%, 70%, 90%, 100%, 120% capacity
- [ ] Đo cả latency và throughput đồng thời — quan sát điểm bão hòa
- [ ] So sánh P50 vs P99 — nếu chênh > 10× thì cần tìm nguyên nhân tail

---

## Tài liệu tham khảo

1. Little, J.D.C. (1961). "A Proof for the Queuing Formula: L = λW"
2. Harchol-Balter, M. "Performance Modeling and Design of Computer Systems: Queueing Theory in Action"
3. Dean, J. & Barroso, L.A. (2013). "The Tail at Scale" — Communications of the ACM
4. Gil Tene. "How NOT to Measure Latency" — Strange Loop Conference
5. Dunning, T. & Ertl, O. "Computing Extremely Accurate Quantiles Using t-Digests"
