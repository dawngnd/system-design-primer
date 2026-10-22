# Denormalization trong SQL — Phân tích chuyên sâu

## 1. Bối cảnh: Normalization là gì?

Để hiểu Denormalization, trước hết phải hiểu **Normalization** — quy trình thiết kế schema để **loại bỏ dữ liệu trùng lặp** (data redundancy) và **đảm bảo tính toàn vẹn** (data integrity).

### Các dạng chuẩn (Normal Forms)

```
  1NF ──→ 2NF ──→ 3NF ──→ BCNF ──→ 4NF ──→ 5NF
  ───────────────────────────────────────────────→
          Càng chuẩn hóa cao → càng ít redundancy
                               → càng nhiều bảng
                               → càng nhiều JOIN
```

| Dạng chuẩn | Yêu cầu cốt lõi | Ví dụ vi phạm |
|------------|-------------------|---------------|
| **1NF** | Mỗi cột chứa giá trị **atomic** (không lặp, không list) | Cột `phone` chứa `"0901,0902"` |
| **2NF** | 1NF + mọi cột non-key phụ thuộc **toàn bộ** primary key | Bảng `(order_id, product_id, product_name)` — `product_name` chỉ phụ thuộc `product_id` |
| **3NF** | 2NF + không có **transitive dependency** (non-key → non-key) | Bảng `(emp_id, dept_id, dept_name)` — `dept_name` phụ thuộc `dept_id`, không phụ thuộc trực tiếp `emp_id` |

### Ví dụ chuẩn hóa đến 3NF

**Trước (1 bảng vi phạm 3NF):**

```sql
CREATE TABLE orders (
    order_id      INT PRIMARY KEY,
    customer_id   INT,
    customer_name VARCHAR(100),   -- ❌ trùng lặp: phụ thuộc customer_id
    customer_city VARCHAR(50),    -- ❌ trùng lặp: phụ thuộc customer_id
    product_id    INT,
    product_name  VARCHAR(100),   -- ❌ trùng lặp: phụ thuộc product_id
    product_price DECIMAL(10,2),  -- ❌ trùng lặp: phụ thuộc product_id
    quantity      INT,
    order_date    DATE
);
```

**Sau khi chuẩn hóa (3NF) — tách thành 3 bảng:**

```sql
CREATE TABLE customers (
    customer_id   INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_city VARCHAR(50)
);

CREATE TABLE products (
    product_id    INT PRIMARY KEY,
    product_name  VARCHAR(100),
    product_price DECIMAL(10,2)
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    product_id  INT REFERENCES products(product_id),
    quantity    INT,
    order_date  DATE
);
```

> ✅ **Ưu điểm**: Không trùng lặp data. Đổi tên customer → chỉ update 1 chỗ.
>
> ❌ **Nhược điểm**: Muốn hiển thị "đơn hàng + tên khách + tên sản phẩm" → cần **JOIN 3 bảng**.

---

## 2. Denormalization là gì?

> **Denormalization = cố ý thêm dữ liệu trùng lặp (redundancy) trở lại schema đã chuẩn hóa, nhằm tăng tốc đọc (read performance) bằng cách giảm hoặc loại bỏ JOIN.**

```
                    Normalization
          ┌──────────────────────────────┐
          │                              │
  Redundant Data ──────────────→  Clean, Normalized Schema
          │                              │
          │         Denormalization       │
          │  ◄─────────────────────────  │
          │   (thêm lại redundancy       │
          │    có kiểm soát)             │
          └──────────────────────────────┘
```

> [!IMPORTANT]
> Denormalization **KHÔNG phải** là "không normalize". Nó là bước **có chủ đích** sau khi đã normalize — bạn hiểu rõ trade-off và chấp nhận redundancy để đổi lấy performance.

---

## 3. Tại sao cần Denormalize?

### Vấn đề cốt lõi của schema chuẩn hóa ở scale lớn

```sql
-- Query "đơn giản" nhưng cần JOIN 5 bảng
SELECT o.order_id, c.name, p.product_name, cat.category_name, s.shipper_name
FROM orders o
JOIN customers c    ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p     ON oi.product_id = p.product_id
JOIN categories cat ON p.category_id = cat.category_id
JOIN shippers s     ON o.shipper_id = s.shipper_id
WHERE o.order_date > '2025-01-01';
```

**Với 10 triệu orders, mỗi JOIN nhân chi phí:**

| Yếu tố | Ảnh hưởng |
|---------|-----------|
| **Disk I/O** | Mỗi bảng có thể nằm ở vị trí khác trên disk → random I/O |
| **CPU** | Hash join hoặc merge join tốn CPU để so khớp |
| **Memory** | Intermediate result sets chiếm RAM |
| **Lock contention** | Đọc nhiều bảng → giữ lock lâu hơn |
| **Query plan complexity** | Optimizer phải chọn thứ tự JOIN tối ưu — dễ sai với nhiều bảng |

**Benchmark thực tế (minh họa):**

```
Schema 3NF (5-table JOIN):     ~120ms / query   ← quá chậm cho web
Schema denormalized (1 bảng):  ~8ms / query      ← chấp nhận được
```

---

## 4. Các kỹ thuật Denormalization

### 4.1. Thêm cột dư thừa (Redundant Column)

**Kỹ thuật phổ biến nhất.** Copy cột từ bảng liên quan vào bảng chính để tránh JOIN.

```sql
-- TRƯỚC: cần JOIN để lấy customer_name
SELECT o.order_id, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;

-- SAU: thêm cột trực tiếp
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(100);

-- Query không cần JOIN nữa
SELECT order_id, customer_name FROM orders;
```

| Khi nào dùng | Khi nào tránh |
|--------------|---------------|
| Cột ít thay đổi (tên sản phẩm, tên danh mục) | Cột thay đổi thường xuyên (giá, trạng thái) |
| Query đọc rất nhiều, write ít | Write-heavy workload |

---

### 4.2. Cột tính toán trước (Pre-computed / Derived Column)

Lưu kết quả tính toán thay vì tính mỗi lần query.

```sql
-- TRƯỚC: tính tổng tiền mỗi lần
SELECT o.order_id, SUM(oi.price * oi.quantity) AS total
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id;

-- SAU: lưu sẵn
ALTER TABLE orders ADD COLUMN order_total DECIMAL(12,2);

-- Cập nhật khi thêm/sửa order_item
UPDATE orders SET order_total = (
    SELECT SUM(price * quantity) FROM order_items WHERE order_id = 1001
) WHERE order_id = 1001;
```

**Các ví dụ thực tế:**

```sql
-- Bảng products: lưu sẵn rating trung bình
ALTER TABLE products ADD COLUMN avg_rating DECIMAL(3,2);
ALTER TABLE products ADD COLUMN review_count INT DEFAULT 0;

-- Bảng users: lưu sẵn số bạn bè
ALTER TABLE users ADD COLUMN friend_count INT DEFAULT 0;

-- Bảng posts: lưu sẵn số comment
ALTER TABLE posts ADD COLUMN comment_count INT DEFAULT 0;
```

---

### 4.3. Bảng tổng hợp (Summary / Aggregate Table)

Tạo bảng riêng chứa data đã tổng hợp cho báo cáo.

```sql
-- Thay vì chạy query nặng mỗi lần xem dashboard:
SELECT DATE(order_date) AS day,
       COUNT(*) AS total_orders,
       SUM(order_total) AS revenue
FROM orders
WHERE order_date >= '2025-01-01'
GROUP BY DATE(order_date);   -- scan hàng triệu rows

-- Tạo summary table, cập nhật định kỳ hoặc real-time
CREATE TABLE daily_sales_summary (
    summary_date  DATE PRIMARY KEY,
    total_orders  INT,
    total_revenue DECIMAL(15,2),
    avg_order_value DECIMAL(10,2),
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Phù hợp cho:** Dashboard, báo cáo, analytics — nơi data không cần real-time tuyệt đối.

---

### 4.4. Hợp nhất bảng (Table Merging)

Gộp hai bảng có quan hệ 1:1 hoặc 1:N (ít N) thành một.

```sql
-- TRƯỚC: 2 bảng quan hệ 1:1
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email   VARCHAR(255)
);
CREATE TABLE user_profiles (
    user_id  INT PRIMARY KEY REFERENCES users(user_id),
    bio      TEXT,
    avatar   VARCHAR(255),
    location VARCHAR(100)
);

-- SAU: gộp thành 1 bảng
CREATE TABLE users (
    user_id  INT PRIMARY KEY,
    email    VARCHAR(255),
    bio      TEXT,
    avatar   VARCHAR(255),
    location VARCHAR(100)
);
```

---

### 4.5. Lưu dạng JSON / Array (Embedded Data)

Nhúng data con trực tiếp vào cột JSON — kiểu hybrid SQL + NoSQL.

```sql
-- TRƯỚC: bảng riêng cho tags
CREATE TABLE article_tags (
    article_id INT,
    tag        VARCHAR(50),
    PRIMARY KEY (article_id, tag)
);

-- SAU: nhúng vào cột JSON (MySQL 5.7+ / PostgreSQL 9.4+)
ALTER TABLE articles ADD COLUMN tags JSON;

-- Lưu: UPDATE articles SET tags = '["sql","database","performance"]' WHERE id = 1;
-- Đọc: SELECT * FROM articles WHERE JSON_CONTAINS(tags, '"sql"');
```

> [!WARNING]
> JSON column **không thay thế** quan hệ phức tạp. Chỉ phù hợp khi data con ít, ít thay đổi, và thường đọc cùng bản ghi cha.

---

### 4.6. Materialized View

Một dạng denormalization **tự động** do database quản lý (PostgreSQL hỗ trợ, MySQL không native).

```sql
-- PostgreSQL
CREATE MATERIALIZED VIEW order_details_mv AS
SELECT o.order_id, o.order_date,
       c.customer_name, c.customer_city,
       p.product_name, p.product_price,
       oi.quantity,
       (p.product_price * oi.quantity) AS line_total
FROM orders o
JOIN customers c    ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p     ON oi.product_id = p.product_id;

-- Refresh khi cần (không tự động)
REFRESH MATERIALIZED VIEW order_details_mv;

-- Query siêu nhanh — không JOIN
SELECT * FROM order_details_mv WHERE customer_city = 'Hanoi';
```

---

## 5. Trade-off: Cái giá của Denormalization

```
                READ PERFORMANCE ↑
                        │
                        │    ╔══════════════════╗
                        │    ║  DENORMALIZED     ║
                        │    ║  (nhanh đọc,      ║
                        │    ║   chậm ghi,       ║
                        │    ║   risk mất đồng bộ)║
                        │    ╚══════════════════╝
                        │
  WRITE COMPLEXITY ◄────┼────► DATA CONSISTENCY
                        │
                        │    ╔══════════════════╗
                        │    ║  NORMALIZED       ║
                        │    ║  (chậm đọc,       ║
                        │    ║   dễ ghi,         ║
                        │    ║   data sạch)      ║
                        │    ╚══════════════════╝
                        │
                WRITE PERFORMANCE ↑
```

### Chi tiết trade-off

| Được (Pros) | Mất (Cons) |
|-------------|-----------|
| ⚡ Giảm/loại bỏ JOIN → query nhanh hơn | 💾 **Tốn thêm disk** do data trùng lặp |
| 📉 Query plan đơn giản hơn → optimizer ít sai | 🔄 **Write chậm hơn** — phải update nhiều chỗ |
| 🧠 Code SELECT đơn giản, dễ đọc | 🐛 **Risk data inconsistency** — quên update 1 chỗ → data sai |
| 📊 Phù hợp **read-heavy** workload (90% read) | 🏗️ **Schema cứng hơn** — thay đổi business logic → sửa nhiều chỗ |
| 🚀 Dễ scale đọc (cache, replica) | 🧪 **Test phức tạp hơn** — phải verify data đồng bộ |

---

## 6. Giữ data đồng bộ khi Denormalize — Chiến lược thực tế

Đây là **vấn đề khó nhất** của denormalization. Có 4 cách chính:

### 6.1. Application-level sync

```java
// Khi update customer name → phải update cả bảng orders
@Transactional
public void updateCustomerName(int customerId, String newName) {
    customerRepo.updateName(customerId, newName);
    orderRepo.updateCustomerName(customerId, newName);  // ← dễ quên!
}
```

> ❌ Dễ quên, dễ bug. Không khuyến nghị ở scale lớn.

### 6.2. Database Trigger

```sql
-- Tự động cập nhật orders khi customer đổi tên
CREATE TRIGGER sync_customer_name
AFTER UPDATE ON customers
FOR EACH ROW
BEGIN
    IF OLD.customer_name != NEW.customer_name THEN
        UPDATE orders
        SET customer_name = NEW.customer_name
        WHERE customer_id = NEW.customer_id;
    END IF;
END;
```

> ✅ Đảm bảo đồng bộ ở tầng DB.
> ❌ Trigger ẩn logic, khó debug, có thể gây cascade chậm.

### 6.3. Event-driven / CDC (Change Data Capture)

```
customers table ──(CDC)──→ Kafka/Debezium ──→ Consumer ──→ Update orders table
```

> ✅ Decoupled, scale tốt, có audit trail.
> ⚠️ **Eventual consistency** — có độ trễ giữa lúc đổi và lúc đồng bộ.

### 6.4. Batch job / Cron

```sql
-- Chạy mỗi 5 phút: sync lại tất cả
UPDATE orders o
JOIN customers c ON o.customer_id = c.customer_id
SET o.customer_name = c.customer_name
WHERE o.customer_name != c.customer_name;
```

> ✅ Đơn giản, dễ implement.
> ❌ Data stale trong khoảng giữa 2 lần chạy.

---

## 7. Ví dụ thực tế: E-commerce Order Page

### Yêu cầu hiển thị

```
┌─────────────────────────────────────────┐
│  Đơn hàng #10042                        │
│  Ngày: 2025-10-08                       │
│  Khách: Nguyễn Văn A — Hà Nội          │
│  ─────────────────────────────────────  │
│  Sản phẩm         SL    Đơn giá  Thành │
│  iPhone 15 Pro      1   28.990   28.990│
│  AirPods Pro 2      2    5.990   11.980│
│  ─────────────────────────────────────  │
│  Tổng cộng:                     40.970 │
│  Trạng thái: Đang giao                  │
│  Đánh giá TB sản phẩm: ⭐ 4.7          │
└─────────────────────────────────────────┘
```

### Schema chuẩn hóa (3NF) — cần 4 JOIN

```sql
SELECT o.order_id, o.order_date, o.status,
       c.customer_name, c.city,
       p.product_name, p.price, p.avg_rating,
       oi.quantity, (oi.quantity * p.price) AS line_total
FROM orders o
JOIN customers c    ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p     ON oi.product_id = p.product_id
WHERE o.order_id = 10042;
```

### Schema denormalized — 0 JOIN

```sql
CREATE TABLE denorm_order_details (
    order_id       INT,
    order_date     DATE,
    status         VARCHAR(20),
    customer_name  VARCHAR(100),   -- từ customers
    customer_city  VARCHAR(50),    -- từ customers
    product_name   VARCHAR(200),   -- từ products
    unit_price     DECIMAL(10,2),  -- từ products (snapshot tại thời điểm mua)
    quantity       INT,            -- từ order_items
    line_total     DECIMAL(12,2),  -- pre-computed
    product_rating DECIMAL(3,2),   -- từ products
    PRIMARY KEY (order_id, product_name)
);

-- Query chỉ cần:
SELECT * FROM denorm_order_details WHERE order_id = 10042;
```

> [!TIP]
> Trong e-commerce, `unit_price` trong đơn hàng **nên** là snapshot (giá tại lúc mua), không reference từ bảng products. Đây vừa là denormalization vừa là **business requirement** — giá sản phẩm có thể thay đổi nhưng đơn hàng phải giữ giá cũ.

---

## 8. Decision Framework: Có nên Denormalize không?

```
                    Query có JOIN nặng ≥ 3 bảng?
                              │
                     ┌────── YES ──────┐
                     │                 │
              Workload read-heavy      Workload write-heavy
              (>= 80% read)?          (>= 50% write)?
                     │                 │
                    YES               YES
                     │                 │
            ┌────────┘                 └──→ ❌ KHÔNG denormalize
            │                               (focus vào indexing,
            │                                partitioning, caching)
            ▼
    Data thay đổi thường xuyên
    ở bảng source?
            │
     ┌──── YES ────┐────── NO ──────┐
     │              │                │
  Dùng Trigger   Dùng CDC       ✅ Denormalize
  hoặc CDC       (Debezium)     trực tiếp
  để sync        + eventual     (rất an toàn)
                 consistency
```

### Quy tắc ngón tay cái

| Tình huống | Quyết định |
|-----------|-----------|
| API endpoint đọc bởi hàng triệu user/ngày | ✅ Denormalize |
| Admin dashboard nội bộ, vài trăm query/ngày | ❌ Giữ normalized |
| Giá sản phẩm trong đơn hàng (cần snapshot) | ✅ Denormalize (business requirement) |
| Data tài chính cần audit trail chính xác | ⚠️ Cẩn thận — giữ bảng gốc làm source of truth |
| Reporting / Analytics | ✅ Summary table hoặc Materialized View |
| Full-text search kết hợp nhiều bảng | ✅ Denormalize vào search index (Elasticsearch) |

---

## 9. Tổng kết

```
  ┌─────────────────────────────────────────────────────────────┐
  │              SPECTRUM OF DATA MODELING                       │
  │                                                             │
  │  Fully           3NF            Selective         Fully     │
  │  Normalized ◄──────────────────────────────► Denormalized   │
  │                                                             │
  │  ✅ Data sạch     ⭐ Sweet spot   ✅ Read nhanh   ❌ Chaos   │
  │  ❌ Read chậm     cho hầu hết    ⚠️ Write phức   ❌ Data    │
  │                   ứng dụng        tạp hơn         không     │
  │                                                   tin cậy   │
  └─────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Normalize trước, denormalize sau.** Luôn bắt đầu với schema chuẩn hóa, đo lường performance thực tế, xác định bottleneck cụ thể, rồi mới denormalize **đúng chỗ cần thiết**. Denormalization không có kiểm soát = technical debt.
