# JOIN Algorithms — Tổng hợp toàn bộ thuật toán JOIN trong SQL

> **Tổng cộng: ~18 thuật toán / biến thể**, chia thành 5 nhóm.
>
> Database optimizer **tự chọn** thuật toán phù hợp dựa trên kích thước bảng, index có sẵn, thống kê data, và tài nguyên hệ thống. Hiểu chúng giúp đọc `EXPLAIN` plan và tối ưu query.

---

## Mục lục

- [Bức tranh toàn cảnh](#bức-tranh-toàn-cảnh)
- [Nhóm 1 — Nested Loop Join (4 loại)](#nhóm-1--nested-loop-join-4-loại)
- [Nhóm 2 — Hash Join (4 loại)](#nhóm-2--hash-join-4-loại)
- [Nhóm 3 — Sort-Merge Join (2 loại)](#nhóm-3--sort-merge-join-2-loại)
- [Nhóm 4 — Thuật toán chuyên biệt (5 loại)](#nhóm-4--thuật-toán-chuyên-biệt-5-loại)
- [Nhóm 5 — JOIN trong hệ phân tán (3 loại)](#nhóm-5--join-trong-hệ-phân-tán-3-loại)
- [Bảng so sánh tổng hợp Big 3](#bảng-so-sánh-tổng-hợp-big-3)
- [Cách xem DB chọn thuật toán nào](#cách-xem-db-chọn-thuật-toán-nào)
- [Tổng kết](#tổng-kết)

---

## Bức tranh toàn cảnh

```
                            JOIN Algorithms
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
       Nested Loop            Hash Join           Sort-Merge
        (Big 3)                (Big 3)              (Big 3)
            │                     │                     │
       ┌────┼────┐           ┌────┼────┐                │
       │    │    │           │    │    │                │
    Simple Index Block    Simple Grace Recursive    Index-assisted
     NLJ    NLJ   NLJ    Hash   Hash  Hash         Merge
                   │      Join   Join  Join
                   │                    │
                  BKA              Parallel
                                  Hash Join

                  ┌──────────────────────────────┐
                  │      Chuyên biệt (5 loại)     │
                  │  Semi · Anti · Lateral/Apply  │
                  │  Bitmap · Adaptive            │
                  └──────────────────────────────┘

                  ┌──────────────────────────────┐
                  │     Distributed (3 loại)      │
                  │  Broadcast · Shuffle          │
                  │  Colocated                    │
                  └──────────────────────────────┘
```

---

## Nhóm 1 — Nested Loop Join (4 loại)

Ý tưởng cốt lõi: **vòng lặp lồng nhau** — với mỗi row bảng ngoài, duyệt bảng trong để tìm match.

### 1.1. Simple Nested Loop Join

**Cơ chế:**

```
for each row_a in Table_A:            ← outer loop
    for each row_b in Table_B:        ← inner loop (full scan mỗi lần)
        if row_a.key == row_b.key:
            output (row_a, row_b)
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Độ phức tạp** | `O(N × M)` — brute force, tệ nhất |
| **RAM cần** | Rất ít — chỉ giữ 1 row tại mỗi thời điểm |
| **Khi nào dùng** | Hầu như không — chỉ khi không có index, data rất nhỏ |
| **Ưu điểm** | Đơn giản, không cần chuẩn bị |
| **Nhược điểm** | Cực chậm với data lớn. 10K × 10K = 100 triệu phép so sánh |

---

### 1.2. Index Nested Loop Join

**Cơ chế:**

```
for each row_a in Table_A:                 ← outer loop
    row_b = Index_B.seek(row_a.key)        ← index lookup (không full scan!)
    if found:
        output (row_a, row_b)
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Độ phức tạp** | `O(N × log M)` — mỗi lookup qua B-Tree index là `O(log M)` |
| **Yêu cầu** | Inner table **phải có index** trên cột JOIN |
| **Khi nào DB chọn** | Outer table nhỏ (đã filter), inner table có index |
| **Ưu điểm** | Rất nhanh khi outer table nhỏ; trả kết quả streaming (tốt cho `LIMIT`) |
| **Nhược điểm** | Outer table lớn → vẫn chậm vì N lần index seek |

```sql
-- Ví dụ DB chọn Index NLJ:
SELECT * FROM orders o                      -- 50 rows sau WHERE filter
JOIN customers c ON o.customer_id = c.customer_id  -- index trên customer_id
WHERE o.order_date = '2025-10-08';
-- → 50 lần index seek vào customers, rất nhanh
```

---

### 1.3. Block Nested Loop Join (BNL)

**Cơ chế:**

```
Đọc BLOCK rows (ví dụ 1000 rows) từ Table_A vào Join Buffer
    ↓
Scan Table_B 1 lần, so khớp với TẤT CẢ rows trong buffer
    ↓
Đọc block tiếp theo từ Table_A, lặp lại
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Độ phức tạp** | Vẫn `O(N × M)` nhưng **giảm số lần scan** inner table |
| **Cải tiến gì** | Simple NLJ scan inner M lần, BNL chỉ scan `⌈N/block_size⌉` lần |
| **Khi nào DB chọn** | Không có index trên inner table, data vừa phải |
| **Database** | MySQL < 8.0.18 dùng **rất nhiều** (vì không có Hash Join) |
| **Cấu hình** | MySQL: `join_buffer_size` (default 256KB) |

```sql
-- MySQL EXPLAIN output:
-- "Using join buffer (Block Nested Loop)"
```

---

### 1.4. Batched Key Access (BKA)

**Cơ chế:**

```
Gom BATCH key từ outer table vào buffer
    ↓
Sắp xếp các key theo thứ tự disk (Multi-Range Read - MRR)
    ↓
Batch lookup vào index inner table (sequential I/O thay vì random)
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Cải tiến gì** | Kết hợp BNL + Index NLJ + **MRR** → giảm random disk I/O |
| **Khi nào dùng** | Inner table có index, outer table đủ lớn để batch có ý nghĩa |
| **Database** | MySQL specific. Phải bật: `SET optimizer_switch='mrr=on,batched_key_access=on';` |
| **Ưu điểm** | Biến random I/O thành sequential I/O → nhanh hơn nhiều trên HDD |

---

## Nhóm 2 — Hash Join (4 loại)

Ý tưởng cốt lõi: **xây hash table** từ bảng nhỏ, rồi dùng bảng lớn **probe** vào hash table.

### 2.1. Simple (In-Memory) Hash Join

**Cơ chế:**

```
Bước 1 — BUILD:
    Đọc bảng nhỏ hơn (build table)
    Tạo hash table trong RAM:  hash(key) → row

Bước 2 — PROBE:
    Đọc từng row bảng lớn (probe table)
    Tính hash(key) → tra hash table → match → output
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Độ phức tạp** | `O(N + M)` — **nhanh nhất** cho equality join |
| **RAM cần** | Đủ chứa hash table của bảng nhỏ (= kích thước bảng nhỏ × overhead) |
| **Yêu cầu** | Chỉ dùng cho **equality join** (`=`), không dùng cho range (`>`, `<`) |
| **Khi nào DB chọn** | 2 bảng lớn, không index trên cột JOIN, hash table vừa RAM |
| **Database** | PostgreSQL ✅, MySQL 8.0.18+ ✅, Oracle ✅, SQL Server ✅ |

```sql
-- Ví dụ DB chọn Hash Join:
SELECT * FROM orders o                  -- 10 triệu rows
JOIN order_items oi ON o.order_id = oi.order_id  -- không index
-- → Build hash table từ orders (nhỏ hơn), probe bằng order_items
```

---

### 2.2. Grace Hash Join

**Vấn đề:** Hash table **không vừa RAM** → không thể dùng Simple Hash Join.

**Cơ chế:**

```
Bước 1 — PARTITION:
    Dùng hash function h1 chia CẢ 2 bảng thành K partition
    Ghi mỗi partition ra disk
    
    Table_A → [A₁, A₂, ..., Aₖ]  (trên disk)
    Table_B → [B₁, B₂, ..., Bₖ]  (trên disk)

Bước 2 — JOIN từng cặp partition:
    for i = 1 to K:
        Load Aᵢ vào RAM → build hash table
        Load Bᵢ → probe
        Output matches
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Tại sao hoạt động** | Mỗi partition nhỏ hơn bảng gốc → vừa RAM |
| **Độ phức tạp** | `O(N + M)` + chi phí I/O ghi/đọc partition |
| **Khi nào dùng** | Data rất lớn, RAM không đủ cho Simple Hash Join |
| **Nhược điểm** | I/O ghi ra disk + đọc lại → chậm hơn in-memory hash join |
| **Database** | Oracle, PostgreSQL (tự động khi `work_mem` không đủ) |

---

### 2.3. Recursive (Hybrid) Hash Join

**Cơ chế:**

```
Giống Grace Hash nhưng:
  - Partition nào vừa RAM → giữ luôn, join ngay (không ghi disk)
  - Partition nào quá lớn → ghi disk, rồi hash lại lần nữa (recursive)
      với hash function h2 khác → chia nhỏ hơn → join
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Cải tiến gì** | Tối ưu Grace Hash — tận dụng RAM tối đa, giảm I/O |
| **Khi nào dùng** | Data skewed (một số key xuất hiện nhiều hơn → partition lệch) |
| **Database** | Oracle, DB2. PostgreSQL dùng biến thể tương tự |

---

### 2.4. Parallel Hash Join

**Cơ chế:**

```
Thread 1 → Build hash table partition 1 → Probe partition 1
Thread 2 → Build hash table partition 2 → Probe partition 2
...
Thread P → Build hash table partition P → Probe partition P
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Cải tiến gì** | Chia công việc ra nhiều CPU core → **tăng throughput tuyến tính** |
| **Khi nào dùng** | Server nhiều core, query đủ lớn để justify overhead song song |
| **Database** | PostgreSQL 11+ (parallel hash join), Oracle, SQL Server |
| **Cấu hình PostgreSQL** | `max_parallel_workers_per_gather`, `min_parallel_table_scan_size` |

```sql
-- PostgreSQL EXPLAIN output:
-- Parallel Hash Join
--   -> Parallel Seq Scan on orders
--   -> Parallel Hash
--        -> Seq Scan on customers
```

---

## Nhóm 3 — Sort-Merge Join (2 loại)

Ý tưởng cốt lõi: **sắp xếp** cả 2 bảng theo cột JOIN, rồi **duyệt song song** ghép kết quả.

### 3.1. Simple Sort-Merge Join

**Cơ chế:**

```
Bước 1 — SORT:
    Sort Table_A theo join_key
    Sort Table_B theo join_key

Bước 2 — MERGE:
    pointer_a = đầu Table_A
    pointer_b = đầu Table_B
    
    while pointer_a < end AND pointer_b < end:
        if A[pointer_a].key == B[pointer_b].key:
            output match, advance cả hai
        elif A[pointer_a].key < B[pointer_b].key:
            advance pointer_a
        else:
            advance pointer_b
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Độ phức tạp** | Sort: `O(N log N + M log M)` + Merge: `O(N + M)` |
| **Khi nào DB chọn** | Query có `ORDER BY` trên cột JOIN; hoặc cần join nhiều bảng cùng key |
| **Ưu điểm** | Output đã **sorted** sẵn → không cần sort lại cho `ORDER BY` |
| **Ưu điểm** | Hỗ trợ **range join** (`>=`, `BETWEEN`) — Hash Join không làm được |
| **Nhược điểm** | Chi phí sort cao nếu data chưa sort |
| **Database** | PostgreSQL ✅, Oracle ✅, SQL Server ✅, MySQL ❌ (không hỗ trợ) |

```sql
-- Ví dụ DB chọn Sort-Merge:
SELECT * FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY e.dept_id;  -- output cần sort → merge join cho "free" sort
```

---

### 3.2. Index-assisted Merge Join

**Cơ chế:**

```
Data đã sort sẵn nhờ clustered index / B-Tree
    → BỎ QUA bước Sort
    → Chỉ chạy bước Merge: O(N + M)
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Độ phức tạp** | `O(N + M)` — bằng Hash Join nhưng **không cần RAM cho hash table** |
| **Yêu cầu** | Cả 2 bảng có index trên cột JOIN (clustered tốt nhất) |
| **Khi nào dùng** | Data đã sort nhờ index → biến merge join thành thuật toán tối ưu nhất |
| **Ưu điểm** | Nhanh, ít RAM, output sorted, hỗ trợ range join |

---

## Nhóm 4 — Thuật toán chuyên biệt (5 loại)

Không phải thuật toán JOIN "mới" mà là **chiến lược tối ưu** cho các trường hợp đặc biệt.

### 4.1. Semi Join

**Ngữ cảnh:** Chỉ cần biết **"có tồn tại match hay không"**, không cần data từ bảng phải.

```sql
-- SQL trigger Semi Join:
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);

-- Hoặc:
SELECT * FROM customers c
WHERE c.customer_id IN (SELECT customer_id FROM orders);
```

**Cơ chế:**

```
for each row in customers:
    if ANY match found in orders:     ← dừng ngay khi tìm thấy 1 match
        output row
        (KHÔNG tiếp tục tìm match khác)
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Khác gì INNER JOIN** | Không trả duplicate — customer có 100 orders vẫn chỉ trả 1 row |
| **Tối ưu** | Dừng sớm (short-circuit) khi tìm thấy match đầu tiên |
| **Database** | Tất cả DB lớn đều hỗ trợ — optimizer tự nhận diện pattern |

---

### 4.2. Anti Join

**Ngữ cảnh:** Ngược Semi Join — trả row **KHÔNG có match**.

```sql
-- SQL trigger Anti Join:
SELECT * FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);

-- Hoặc:
SELECT * FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

**Cơ chế:**

```
for each row in customers:
    if NO match found in orders:
        output row
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Use case** | Tìm khách chưa mua hàng, sản phẩm chưa bán, orphan records |
| **Tối ưu** | Dừng sớm khi tìm thấy match (→ skip row ngay) |
| **Lưu ý** | `NOT IN` + NULL = **kết quả sai**! Luôn dùng `NOT EXISTS` an toàn hơn |

---

### 4.3. Lateral Join / CROSS APPLY

**Ngữ cảnh:** Với mỗi row bảng trái, chạy một **subquery phụ thuộc vào row đó**.

```sql
-- PostgreSQL: LATERAL
SELECT c.customer_name, recent.*
FROM customers c
CROSS JOIN LATERAL (
    SELECT order_id, order_date, total
    FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY order_date DESC
    LIMIT 3                          -- top 3 đơn hàng gần nhất MỖI khách
) recent;

-- SQL Server: CROSS APPLY (cú pháp tương đương)
SELECT c.customer_name, recent.*
FROM customers c
CROSS APPLY (
    SELECT TOP 3 order_id, order_date, total
    FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY order_date DESC
) recent;
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Khác gì JOIN thường** | Subquery **tham chiếu** cột từ bảng trái → không thể flatten thành hash/merge |
| **Use case** | Top-N per group, unpivot, table-valued functions |
| **Database** | PostgreSQL 9.3+ (`LATERAL`), SQL Server (`CROSS APPLY`/`OUTER APPLY`), Oracle 12c+ |
| **MySQL** | 8.0.14+ hỗ trợ `LATERAL` derived tables |

---

### 4.4. Bitmap Join

**Cơ chế:**

```
Bước 1: Scan bảng dimension → tạo bitmap (mảng bit) đánh dấu row match
        Ví dụ: customers ở 'Hanoi' → bitmap [1,0,0,1,1,0,...]

Bước 2: AND/OR nhiều bitmap từ nhiều bảng dimension

Bước 3: Dùng bitmap cuối cùng → fetch chỉ những row match từ bảng fact
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Use case** | Star schema (data warehouse) — 1 fact table + nhiều dimension tables |
| **Ưu điểm** | Kết hợp filter từ nhiều dimension rất nhanh (bitwise operation) |
| **Database** | Oracle (Bitmap Join Index), PostgreSQL (BitmapAnd/BitmapOr) |

```sql
-- PostgreSQL EXPLAIN output:
-- BitmapAnd
--   -> Bitmap Index Scan on idx_city
--   -> Bitmap Index Scan on idx_category
-- -> Bitmap Heap Scan on orders
```

---

### 4.5. Adaptive Join

**Cơ chế:**

```
Optimizer tạo plan có "decision point":
    
    Bắt đầu chạy query
        ↓
    Đếm actual rows ở giai đoạn đầu
        ↓
    Nếu rows ít hơn threshold → chuyển sang Nested Loop
    Nếu rows nhiều hơn threshold → chuyển sang Hash Join
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Vấn đề giải quyết** | Optimizer estimate sai → chọn sai thuật toán → query chậm |
| **Cải tiến** | Quyết định **lúc runtime** dựa trên data thực, không phải statistics |
| **Database** | SQL Server 2017+ (Adaptive Join), Oracle 12c+ (Adaptive Plans) |
| **PostgreSQL** | Chưa có native — nhưng có Adaptive Hash Join (tự spill to disk khi cần) |

---

## Nhóm 5 — JOIN trong hệ phân tán (3 loại)

Trong distributed database / data lake, data nằm trên **nhiều node**. Thách thức: data cần JOIN có thể ở **khác node** → phải truyền qua mạng.

### 5.1. Broadcast Join (Map-Side Join)

**Cơ chế:**

```
┌──────────┐     copy toàn bộ      ┌──── Node 1: small_table + big_partition_1 → join
│          │ ──────────────────────→├──── Node 2: small_table + big_partition_2 → join
│  Small   │ ──────────────────────→├──── Node 3: small_table + big_partition_3 → join
│  Table   │ ──────────────────────→└──── Node 4: small_table + big_partition_4 → join
│ (< 200MB)│
└──────────┘
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Khi nào dùng** | 1 bảng **nhỏ** (vừa RAM mỗi node), 1 bảng lớn |
| **Ưu điểm** | Không cần shuffle bảng lớn → rất nhanh |
| **Nhược điểm** | Bảng nhỏ phải vừa RAM; network cost = `small_table_size × num_nodes` |
| **Database** | Spark (`broadcast()` hint), Presto, BigQuery, Trino |

```python
# Spark example:
from pyspark.sql.functions import broadcast
result = big_df.join(broadcast(small_df), "key")
```

---

### 5.2. Shuffle (Repartition) Hash Join

**Cơ chế:**

```
                    SHUFFLE PHASE
Table_A                                         Table_B
┌─────────┐    hash(key) % N                   ┌─────────┐
│ A data   │ ──────────────→  Node 1 ←──────── │ B data   │
│ (all     │ ──────────────→  Node 2 ←──────── │ (all     │
│  nodes)  │ ──────────────→  Node 3 ←──────── │  nodes)  │
└─────────┘                                    └─────────┘
                Cùng key → cùng node
                      ↓
                JOIN LOCAL trên mỗi node
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Khi nào dùng** | Cả 2 bảng **đều lớn**, không broadcast được |
| **Chi phí** | Network I/O cao — phải truyền data qua mạng |
| **Ưu điểm** | Hoạt động với mọi kích thước data |
| **Nhược điểm** | Chậm nhất trong 3 loại distributed join — tránh nếu có thể |
| **Database** | Spark (Sort-Merge Join mặc định), Presto, BigQuery |

---

### 5.3. Colocated Join

**Cơ chế:**

```
Data đã partition sẵn theo CÙNG KEY trên CÙNG NODE
    → JOIN local ngay, KHÔNG cần network transfer

Node 1: users (shard_key=user_id 1-1000)   + orders (shard_key=user_id 1-1000)    → join local
Node 2: users (shard_key=user_id 1001-2000) + orders (shard_key=user_id 1001-2000) → join local
```

| Đặc điểm | Chi tiết |
|-----------|----------|
| **Yêu cầu** | 2 bảng phải **shard/partition theo cùng key** |
| **Ưu điểm** | **Nhanh nhất** — không network overhead, như join trên single node |
| **Nhược điểm** | Ràng buộc thiết kế — phải plan shard key từ đầu |
| **Database** | CockroachDB (interleaved tables), Citus, Vitess, YugabyteDB |
| **Thiết kế tip** | Luôn shard bảng con theo foreign key của bảng cha |

---

## Bảng so sánh tổng hợp Big 3

| | Nested Loop | Hash Join | Sort-Merge Join |
|---|:-----------:|:---------:|:---------------:|
| **Tốt nhất khi** | 1 bảng nhỏ + có index | 2 bảng lớn, không index | Data đã sort sẵn |
| **Độ phức tạp** | `O(N × M)` hoặc `O(N log M)` | `O(N + M)` | `O(N log N + M log M)` |
| **RAM cần** | Rất ít | Nhiều (hash table) | Trung bình (sort buffer) |
| **Cần index?** | Có → nhanh hơn rất nhiều | Không cần | Có → bỏ qua bước sort |
| **Equality join** (`=`) | ✅ | ✅ | ✅ |
| **Range join** (`>`, `<`, `BETWEEN`) | ✅ | ❌ | ✅ |
| **Output sorted?** | ❌ | ❌ | ✅ |
| **Streaming (tốt cho LIMIT)?** | ✅ | ❌ | ❌ |
| **MySQL** | ✅ Mọi version | ✅ 8.0.18+ | ❌ Không hỗ trợ |
| **PostgreSQL** | ✅ | ✅ | ✅ |
| **Oracle** | ✅ | ✅ | ✅ |
| **SQL Server** | ✅ | ✅ | ✅ |

---

## Cách xem DB chọn thuật toán nào

### MySQL

```sql
EXPLAIN FORMAT=JSON
SELECT o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > '2025-01-01';
```

Tìm trong output:
- `"access_type": "ref"` → Index Nested Loop
- `"using_join_buffer (hash join)"` → Hash Join
- `"using_join_buffer (Block Nested Loop)"` → BNL

### PostgreSQL

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > '2025-01-01';
```

Output trực tiếp nói tên thuật toán:

```
Hash Join  (cost=3.25..8.50 rows=100 width=80) (actual time=0.05..0.08 rows=95 loops=1)
  Hash Cond: (o.customer_id = c.customer_id)
  Buffers: shared hit=5
  -> Seq Scan on orders o  (Filter: order_date > '2025-01-01')
  -> Hash
      -> Seq Scan on customers c
```

### Force thuật toán (khi cần test)

```sql
-- PostgreSQL: tắt hash join → force nested loop hoặc merge
SET enable_hashjoin = off;
SET enable_mergejoin = off;

-- MySQL: dùng hint
SELECT /*+ NO_HASH_JOIN(o, c) */ ...
SELECT /*+ HASH_JOIN(o, c) */ ...

-- Oracle: dùng hint
SELECT /*+ USE_NL(o c) */ ...     -- force Nested Loop
SELECT /*+ USE_HASH(o c) */ ...   -- force Hash Join
SELECT /*+ USE_MERGE(o c) */ ...  -- force Sort-Merge
```

---

## Tổng kết

```
 Nhóm                    Số lượng   Thuật toán
─────────────────────────────────────────────────────────────────
 Nested Loop variants       4      Simple, Index, Block, BKA
 Hash Join variants         4      Simple, Grace, Recursive, Parallel
 Sort-Merge variants        2      Simple, Index-assisted
 Chuyên biệt                5      Semi, Anti, Lateral, Bitmap, Adaptive
 Distributed                3      Broadcast, Shuffle, Colocated
─────────────────────────────────────────────────────────────────
 TỔNG CỘNG                ~18
```

> **Ghi nhớ:**
> - Bạn **không cần chọn** thuật toán — **query optimizer tự chọn**.
> - Nhưng hiểu chúng giúp bạn: đọc `EXPLAIN` plan, biết tại sao thêm index thay đổi performance, và hiểu tại sao **denormalization** loại bỏ được chi phí lớn — vì nó loại bỏ toàn bộ 18 thuật toán này khỏi query.
