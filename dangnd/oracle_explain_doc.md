# Oracle EXPLAIN PLAN — Hướng dẫn đọc Execution Plan

> Tổng hợp tất cả các cách xem và phân tích execution plan trong Oracle Database, từ cơ bản đến nâng cao.

---

## Mục lục

- [1. EXPLAIN PLAN + DBMS_XPLAN.DISPLAY](#1-explain-plan--dbms_xplandisplay)
- [2. DBMS_XPLAN.DISPLAY_CURSOR — Actual Statistics](#2-dbms_xplandisplay_cursor--actual-statistics)
- [3. AUTOTRACE — SQL*Plus / SQL Developer](#3-autotrace--sqlplus--sql-developer)
- [4. V$SQL_PLAN — Xem plan trong cache (DBA)](#4-vsql_plan--xem-plan-trong-cache-dba)
- [5. Format options của DBMS_XPLAN](#5-format-options-của-dbms_xplan)
- [6. Real-Time SQL Monitoring](#6-real-time-sql-monitoring)
- [7. Cách đọc Execution Plan](#7-cách-đọc-execution-plan)
- [8. Các Operation thường gặp](#8-các-operation-thường-gặp)
- [9. Hints — Force thuật toán JOIN](#9-hints--force-thuật-toán-join)
- [Tóm tắt nhanh](#tóm-tắt-nhanh)

---

## 1. EXPLAIN PLAN + DBMS_XPLAN.DISPLAY

**Cách phổ biến nhất.** Tạo plan mà **không chạy** query — chỉ xem optimizer dự định làm gì.

```sql
-- Bước 1: Tạo plan
EXPLAIN PLAN FOR
SELECT o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > DATE '2025-01-01';

-- Bước 2: Hiển thị plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

**Output mẫu:**

```
Plan hash value: 1943995688

-------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |              |   950 | 85500 |    25   (4)| 00:00:01 |
|*  1 |  HASH JOIN                   |              |   950 | 85500 |    25   (4)| 00:00:01 |
|*  2 |   TABLE ACCESS FULL          | ORDERS       |   950 | 66500 |    15   (0)| 00:00:01 |
|   3 |   TABLE ACCESS FULL          | CUSTOMERS    |  1000 | 20000 |     9   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - access("O"."CUSTOMER_ID"="C"."CUSTOMER_ID")
   2 - filter("O"."ORDER_DATE">TO_DATE('2025-01-01','YYYY-MM-DD'))
```

### Giải thích các cột

| Cột | Ý nghĩa |
|-----|---------|
| **Id** | Thứ tự operation (đọc từ trong ra ngoài, từ trên xuống dưới) |
| **Operation** | Thuật toán DB chọn (HASH JOIN, NESTED LOOPS, SORT MERGE...) |
| **Name** | Tên bảng hoặc index được truy cập |
| **Rows** | Số row **dự đoán** (estimated) |
| **Bytes** | Lượng data dự đoán |
| **Cost (%CPU)** | Chi phí tương đối — số càng lớn càng tốn. `%CPU` = phần trăm chi phí là CPU (còn lại là I/O) |
| **Time** | Thời gian dự đoán |

### Lưu ý

- `EXPLAIN PLAN` chỉ tạo plan, **không thực thi** query → không có actual statistics.
- Plan lưu vào bảng `PLAN_TABLE` (Oracle tự tạo).
- Có thể đặt tag để phân biệt nhiều plan:

```sql
EXPLAIN PLAN SET STATEMENT_ID = 'my_query_1' FOR
SELECT ...;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY('PLAN_TABLE', 'my_query_1', 'ALL'));
```

---

## 2. DBMS_XPLAN.DISPLAY_CURSOR — Actual Statistics

**Mạnh hơn EXPLAIN PLAN** vì hiển thị **actual rows, actual time** sau khi chạy query thật.

### Cách dùng

```sql
-- Bật gather statistics (cần cho ALLSTATS)
ALTER SESSION SET STATISTICS_LEVEL = ALL;

-- Chạy query thật
SELECT o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > DATE '2025-01-01';

-- Xem plan với actual stats (lần chạy cuối cùng)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

**Output:**

```
-------------------------------------------------------------------------------------------------------------------
| Id  | Operation          | Name      | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  |  OMem |  1Mem |
-------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |           |      1 |        |    947 |00:00:00.02 |     320 |     12 |       |       |
|*  1 |  HASH JOIN         |           |      1 |    950 |    947 |00:00:00.02 |     320 |     12 |  1250K|  1250K|
|*  2 |   TABLE ACCESS FULL| ORDERS    |      1 |    950 |    947 |00:00:00.01 |     210 |      8 |       |       |
|   3 |   TABLE ACCESS FULL| CUSTOMERS |      1 |   1000 |   1000 |00:00:00.01 |     110 |      4 |       |       |
-------------------------------------------------------------------------------------------------------------------
```

### Giải thích các cột bổ sung

| Cột | Ý nghĩa |
|-----|---------|
| **Starts** | Số lần operation được thực thi (quan trọng cho Nested Loop) |
| **E-Rows** | Estimated rows — optimizer dự đoán |
| **A-Rows** | **Actual rows** — thực tế |
| **A-Time** | **Actual time** — thời gian thực tế |
| **Buffers** | Logical I/O (consistent gets + db block gets) |
| **Reads** | Physical disk reads |
| **OMem** | Optimal memory (RAM cần nếu đủ) |
| **1Mem** | One-pass memory (RAM tối thiểu trước khi spill to disk) |

### Phát hiện vấn đề

```
Nếu E-Rows = 950 nhưng A-Rows = 500,000
    → Optimizer estimate SAI → có thể chọn sai thuật toán
    → Cần: DBMS_STATS.GATHER_TABLE_STATS hoặc dynamic sampling
```

### Xem plan bằng SQL_ID cụ thể

```sql
-- Tìm SQL_ID
SELECT sql_id, sql_text
FROM v$sql
WHERE sql_text LIKE '%orders%customers%'
  AND sql_text NOT LIKE '%v$sql%';

-- Xem plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('abc123xyz', 0, 'ALLSTATS LAST'));
```

---

## 3. AUTOTRACE — SQL*Plus / SQL Developer

Cách nhanh nhất trong SQL*Plus hoặc SQL Developer.

### Các chế độ

```sql
-- Chạy query + hiển thị data + plan + statistics
SET AUTOTRACE ON

-- Chỉ xem plan + statistics, KHÔNG hiện data (tiết kiệm output)
SET AUTOTRACE TRACEONLY

-- Chỉ xem plan, không chạy query
SET AUTOTRACE TRACEONLY EXPLAIN

-- Chỉ xem statistics (I/O, memory), không hiện plan
SET AUTOTRACE TRACEONLY STATISTICS

-- Tắt autotrace
SET AUTOTRACE OFF
```

### Output mẫu (TRACEONLY)

```
Execution Plan
----------------------------------------------------------
Plan hash value: 1943995688

-------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |              |   950 | 85500 |    25   (4)| 00:00:01 |
|*  1 |  HASH JOIN                   |              |   950 | 85500 |    25   (4)| 00:00:01 |
|*  2 |   TABLE ACCESS FULL          | ORDERS       |   950 | 66500 |    15   (0)| 00:00:01 |
|   3 |   TABLE ACCESS FULL          | CUSTOMERS    |  1000 | 20000 |     9   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------------


Statistics
----------------------------------------------------------
          0  recursive calls
          0  db block gets
        320  consistent gets        ← logical reads (buffer cache)
         12  physical reads         ← disk reads
          0  redo size
      45230  bytes sent via SQL*Net to client
        520  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
        947  rows processed
```

### Giải thích Statistics

| Metric | Ý nghĩa | Muốn giá trị |
|--------|---------|--------------|
| **recursive calls** | Lệnh SQL nội bộ Oracle chạy thêm (parse, space mgmt) | Thấp |
| **db block gets** | Current-mode reads (thường từ INSERT/UPDATE) | Thấp |
| **consistent gets** | Consistent-mode reads — **logical I/O chính** | **Càng thấp càng tốt** |
| **physical reads** | Đọc từ disk (không có trong buffer cache) | Thấp |
| **redo size** | Bytes ghi vào redo log (cho DML) | 0 cho SELECT |
| **sorts (memory)** | Số phép sort thực hiện trong RAM | OK |
| **sorts (disk)** | Số phép sort phải dùng disk (RAM không đủ) | **Phải = 0** |
| **rows processed** | Số row trả về | Đúng với kỳ vọng |

### Yêu cầu quyền

```sql
-- Cần role PLUSTRACE (DBA grant):
GRANT PLUSTRACE TO your_user;

-- Hoặc tạo PLAN_TABLE nếu chưa có:
@$ORACLE_HOME/rdbms/admin/utlxplan.sql
```

---

## 4. V$SQL_PLAN — Xem plan trong cache (DBA)

Truy cập trực tiếp plan cache — không cần chạy lại query.

### Tìm query trong cache

```sql
-- Tìm theo text
SELECT sql_id, plan_hash_value, executions, elapsed_time/1000000 AS elapsed_sec,
       SUBSTR(sql_text, 1, 100) AS sql_preview
FROM v$sql
WHERE UPPER(sql_text) LIKE '%ORDERS%CUSTOMERS%'
  AND sql_text NOT LIKE '%v$sql%'
ORDER BY elapsed_time DESC;
```

### Xem plan bằng SQL_ID

```sql
-- Cách 1: DBMS_XPLAN (đẹp, dễ đọc)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('abc123xyz', 0, 'TYPICAL'));

-- Cách 2: Query trực tiếp V$SQL_PLAN (linh hoạt hơn)
SELECT id, parent_id, operation, options, object_name, cost, cardinality, bytes
FROM v$sql_plan
WHERE sql_id = 'abc123xyz'
  AND child_number = 0
ORDER BY id;
```

### Xem tất cả plan của cùng 1 query (plan flip)

```sql
-- Một query có thể có NHIỀU plan khác nhau (child cursors)
SELECT child_number, plan_hash_value, executions, 
       buffer_gets, disk_reads, elapsed_time
FROM v$sql
WHERE sql_id = 'abc123xyz';

-- Xem plan cụ thể
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('abc123xyz', 0));  -- child 0
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('abc123xyz', 1));  -- child 1
```

---

## 5. Format options của DBMS_XPLAN

Dùng cho cả `DISPLAY` và `DISPLAY_CURSOR`.

### Các preset format

| Format | Hiển thị | Dùng khi |
|--------|---------|---------|
| `BASIC` | Chỉ operation + object name | Xem nhanh cấu trúc plan |
| `TYPICAL` | Cost, rows, bytes, time, predicate **(mặc định)** | Phân tích thông thường |
| `ALL` | Tất cả thông tin có sẵn | Phân tích chi tiết |
| `ALLSTATS LAST` | Actual statistics lần chạy cuối | **Debug performance** — dùng nhiều nhất |
| `ADVANCED` | Tất cả + outline hints + query block + column projection | Troubleshooting optimizer |

### Các modifier bổ sung

Có thể thêm vào sau preset bằng dấu `+` hoặc `-`:

```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST +PREDICATE +COST +NOTE'));
```

| Modifier | Thêm/bớt gì |
|----------|-------------|
| `+PREDICATE` | Hiện filter/access predicates (WHERE, JOIN condition) |
| `+COST` | Hiện cột Cost |
| `+BYTES` | Hiện cột Bytes |
| `+PARALLEL` | Thông tin parallel execution (PX) |
| `+PARTITION` | Thông tin partition pruning |
| `+PROJECTION` | Cột nào được project (SELECT) ở mỗi bước |
| `+ALIAS` | Query block alias |
| `+REMOTE` | Thông tin remote execution (DB link) |
| `+NOTE` | Ghi chú (dynamic sampling, cardinality feedback, SQL plan baseline…) |
| `+OUTLINE` | Outline hints — **có thể copy để pin plan** |
| `-ROWS` | Ẩn cột Rows |
| `-COST` | Ẩn cột Cost |

### Ví dụ kết hợp

```sql
-- Debug đầy đủ nhất
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 
    'ALLSTATS LAST +PREDICATE +COST +NOTE +OUTLINE'));

-- Xem nhanh chỉ cấu trúc
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'BASIC +PREDICATE'));

-- Xem parallel execution
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST +PARALLEL'));
```

---

## 6. Real-Time SQL Monitoring

Tự động bật cho query chạy **> 5 giây** hoặc **parallel query**. Cung cấp live monitoring.

### Xem report dạng text

```sql
SELECT DBMS_SQLTUNE.REPORT_SQL_MONITOR(
    sql_id       => 'abc123xyz',
    type         => 'TEXT',
    report_level => 'ALL'
) FROM dual;
```

### Xem report dạng HTML (mở trong browser)

```sql
-- Spool ra file HTML
SET LONG 1000000
SET LONGCHUNKSIZE 1000000
SET LINESIZE 1000
SPOOL /tmp/sql_monitor.html

SELECT DBMS_SQLTUNE.REPORT_SQL_MONITOR(
    sql_id       => 'abc123xyz',
    type         => 'ACTIVE',         -- interactive HTML report
    report_level => 'ALL'
) FROM dual;

SPOOL OFF
```

### Xem query đang chạy (không cần sql_id)

```sql
-- Liệt kê tất cả query đang được monitor
SELECT sql_id, status, sql_text, elapsed_time/1000000 AS sec,
       buffer_gets, disk_reads
FROM v$sql_monitor
WHERE status = 'EXECUTING'
ORDER BY elapsed_time DESC;
```

### Yêu cầu

- Oracle Enterprise Edition
- `STATISTICS_LEVEL = TYPICAL` hoặc `ALL`
- Hoặc hint: `/*+ MONITOR */`

```sql
-- Force monitoring cho query ngắn
SELECT /*+ MONITOR */ o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```

---

## 7. Cách đọc Execution Plan

### Quy tắc đọc

```
1. Indent sâu nhất → thực hiện TRƯỚC
2. Cùng mức indent → đọc từ TRÊN xuống DƯỚI  
3. Child operation → gửi kết quả lên Parent operation
```

### Ví dụ đọc plan

```
-------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name          | Rows  | Cost |
-------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |               |  1000 |   45 |      ← 5. Trả kết quả
|   1 |  SORT ORDER BY                |               |  1000 |   45 |      ← 4. Sort
|*  2 |   HASH JOIN                   |               |  1000 |   40 |      ← 3. Hash Join
|   3 |    TABLE ACCESS BY INDEX ROWID| CUSTOMERS     |   100 |   10 |      ← 1. Đọc customers (build)
|*  4 |     INDEX RANGE SCAN          | IDX_CUST_CITY |   100 |    2 |      ← (qua index trước)
|   5 |    TABLE ACCESS FULL          | ORDERS        | 50000 |   25 |      ← 2. Scan orders (probe)
-------------------------------------------------------------------------------------------------

Thứ tự thực thi: 4 → 3 → 5 → 2 → 1 → 0
```

### Biểu đồ luồng data

```
  INDEX RANGE SCAN (idx_cust_city, city='Hanoi')        100 rows
         │
         ▼
  TABLE ACCESS BY ROWID (customers)                     100 rows (build input)
         │
         ├────────────────────┐
         │                    │
         ▼                    ▼
  HASH JOIN  ◄────── TABLE ACCESS FULL (orders)         50,000 rows (probe input)
         │
         ▼                                              1,000 rows matched
  SORT ORDER BY
         │
         ▼                                              1,000 rows sorted
  SELECT STATEMENT (output)
```

---

## 8. Các Operation thường gặp

### Table Access

| Operation | Ý nghĩa | Tốt/xấu |
|-----------|---------|---------|
| `TABLE ACCESS FULL` | Full table scan — đọc toàn bộ bảng | ⚠️ Chậm nếu bảng lớn + chỉ cần ít row |
| `TABLE ACCESS BY INDEX ROWID` | Đọc row qua ROWID (từ index) | ✅ Nhanh cho ít row |
| `TABLE ACCESS BY INDEX ROWID BATCHED` | Batch nhiều ROWID rồi đọc | ✅ Tối ưu hơn version không batch |

### Index Access

| Operation | Ý nghĩa |
|-----------|---------|
| `INDEX UNIQUE SCAN` | Tìm chính xác 1 row qua primary key / unique index |
| `INDEX RANGE SCAN` | Quét range trong index (WHERE col > X, BETWEEN, LIKE 'abc%') |
| `INDEX FULL SCAN` | Đọc toàn bộ index (sorted) — tránh sort |
| `INDEX FAST FULL SCAN` | Đọc toàn bộ index (unsorted, multi-block read) — nhanh hơn full scan |
| `INDEX SKIP SCAN` | Skip leading column của composite index |

### JOIN Operations

| Operation | Thuật toán | Khi nào optimizer chọn |
|-----------|-----------|----------------------|
| `NESTED LOOPS` | Nested Loop Join | 1 bảng nhỏ + có index trên bảng kia |
| `HASH JOIN` | Hash Join | 2 bảng lớn, equality join, không/có index |
| `SORT MERGE JOIN` | Sort-Merge Join | Data đã sort, hoặc range join |
| `HASH JOIN SEMI` | Semi Hash Join | `EXISTS`, `IN (subquery)` |
| `HASH JOIN ANTI` | Anti Hash Join | `NOT EXISTS`, `NOT IN (subquery)` |
| `NESTED LOOPS SEMI` | Semi Nested Loop | `EXISTS` với bảng nhỏ + index |
| `NESTED LOOPS ANTI` | Anti Nested Loop | `NOT EXISTS` với bảng nhỏ + index |

### Sort / Aggregate

| Operation | Ý nghĩa |
|-----------|---------|
| `SORT ORDER BY` | Sort cho `ORDER BY` |
| `SORT GROUP BY` | Sort cho `GROUP BY` |
| `HASH GROUP BY` | Hash-based grouping (thường nhanh hơn sort) |
| `SORT AGGREGATE` | Tính `COUNT`, `SUM`, `MAX`, `MIN` |
| `SORT UNIQUE` | Loại duplicate cho `DISTINCT` |
| `WINDOW SORT` | Sort cho analytic functions (`ROW_NUMBER`, `RANK`…) |

### Khác

| Operation | Ý nghĩa |
|-----------|---------|
| `FILTER` | Áp dụng filter condition |
| `VIEW` | Truy cập view hoặc inline view |
| `UNION-ALL` | Kết hợp kết quả `UNION ALL` |
| `CONCATENATION` | Optimizer rewrite `OR` thành `UNION ALL` |
| `PARTITION RANGE` | Partition pruning — chỉ scan partition cần thiết |
| `PX SEND / PX RECEIVE` | Parallel execution — phân phối data giữa PX slaves |

---

## 9. Hints — Force thuật toán JOIN

Khi optimizer chọn sai, dùng hint để force thuật toán.

### JOIN hints

```sql
-- Force Nested Loop
SELECT /*+ USE_NL(o c) */ o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;

-- Force Hash Join
SELECT /*+ USE_HASH(o c) */ o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;

-- Force Sort-Merge Join
SELECT /*+ USE_MERGE(o c) */ o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;

-- Force thứ tự join (bảng nào là driving table)
SELECT /*+ LEADING(c o) USE_HASH(o) */ o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```

### Access path hints

```sql
-- Force full table scan (bỏ qua index)
SELECT /*+ FULL(o) */ * FROM orders o WHERE order_id = 100;

-- Force dùng index cụ thể
SELECT /*+ INDEX(o idx_order_date) */ * FROM orders o WHERE order_date > SYSDATE - 30;

-- Force index fast full scan
SELECT /*+ INDEX_FFS(o idx_order_date) */ order_date FROM orders o;
```

### Parallel hints

```sql
-- Force parallel execution
SELECT /*+ PARALLEL(o, 4) PARALLEL(c, 4) */ o.*, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```

### Lưu ý quan trọng

> ⚠️ **Hint là biện pháp cuối cùng.** Ưu tiên:
> 1. Cập nhật statistics: `DBMS_STATS.GATHER_TABLE_STATS`
> 2. Thêm/sửa index
> 3. Rewrite query
> 4. Dùng SQL Plan Baseline để pin plan tốt
> 5. **Cuối cùng** mới dùng hint

---

## Tóm tắt nhanh

| Tình huống | Dùng gì |
|-----------|---------|
| Xem nhanh plan, **chưa chạy** query | `EXPLAIN PLAN FOR ...;` → `DBMS_XPLAN.DISPLAY` |
| Xem plan + **actual stats** sau khi chạy | `DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST')` |
| Debug nhanh trong SQL*Plus | `SET AUTOTRACE ON` |
| DBA phân tích query trong cache | `V$SQL` + `DBMS_XPLAN.DISPLAY_CURSOR(sql_id)` |
| Query chạy lâu, cần monitor real-time | `DBMS_SQLTUNE.REPORT_SQL_MONITOR` |
| So sánh estimated vs actual | `E-Rows` vs `A-Rows` trong `ALLSTATS LAST` |
| Force thuật toán JOIN | `/*+ USE_NL */`, `/*+ USE_HASH */`, `/*+ USE_MERGE */` |
