# 第6章: インデックスとパフォーマンス

## 6.1 はじめに

この章では、SQLiteデータベースのパフォーマンスを最適化する方法を学びます。インデックスの仕組みを理解し、適切に活用することで、クエリの実行速度を劇的に改善できます。また、実行計画の分析方法やパフォーマンスチューニングのテクニックについても解説します。

## 6.2 インデックスの基本概念

### インデックスとは

インデックスは、データベーステーブルの検索を高速化するためのデータ構造です。本の索引と同じように、特定のデータを素早く見つけることができます。

```sql
-- テストテーブルの作成
CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    age INTEGER,
    city TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- 大量のテストデータ挿入
WITH RECURSIVE generate_series(value) AS (
    SELECT 1
    UNION ALL
    SELECT value + 1 FROM generate_series WHERE value < 100000
)
INSERT INTO customers (name, email, age, city)
SELECT 
    'Customer' || value,
    'customer' || value || '@example.com',
    20 + (value % 60),
    CASE (value % 5)
        WHEN 0 THEN '東京'
        WHEN 1 THEN '大阪'
        WHEN 2 THEN '名古屋'
        WHEN 3 THEN '福岡'
        ELSE '札幌'
    END
FROM generate_series;
```

### インデックスの効果測定

```sql
-- インデックスなしでの検索（実行計画の確認）
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE city = '東京';

-- 実行時間の測定
.timer on
SELECT COUNT(*) FROM customers WHERE city = '東京';
.timer off

-- インデックスの作成
CREATE INDEX idx_customers_city ON customers(city);

-- インデックスありでの検索
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE city = '東京';

-- 実行時間の再測定
.timer on
SELECT COUNT(*) FROM customers WHERE city = '東京';
.timer off
```

## 6.3 インデックスの種類と構造

### B-Treeインデックス

SQLiteの標準的なインデックスはB-Tree（Balanced Tree）構造を使用します。

```sql
-- 単一列インデックス
CREATE INDEX idx_customers_age ON customers(age);

-- 複合インデックス（複数列）
CREATE INDEX idx_customers_city_age ON customers(city, age);

-- 部分インデックス（条件付き）
CREATE INDEX idx_adult_customers ON customers(age) WHERE age >= 20;

-- 式インデックス
CREATE INDEX idx_customers_email_domain ON customers(
    substr(email, instr(email, '@') + 1)
);
```

### UNIQUEインデックス

```sql
-- UNIQUE制約は自動的にインデックスを作成
CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    sku TEXT UNIQUE,  -- 自動的にインデックスが作成される
    name TEXT,
    price REAL
);

-- 明示的なUNIQUEインデックス
CREATE UNIQUE INDEX idx_products_name ON products(name);

-- 複合UNIQUEインデックス
CREATE UNIQUE INDEX idx_products_composite ON products(sku, name);
```

## 6.4 インデックスの設計原則

### カーディナリティの考慮

```sql
-- カーディナリティ（値の種類）の確認
SELECT 
    'city' AS column_name,
    COUNT(DISTINCT city) AS distinct_values,
    COUNT(*) AS total_rows,
    ROUND(COUNT(DISTINCT city) * 100.0 / COUNT(*), 2) AS selectivity_pct
FROM customers
UNION ALL
SELECT 
    'age',
    COUNT(DISTINCT age),
    COUNT(*),
    ROUND(COUNT(DISTINCT age) * 100.0 / COUNT(*), 2)
FROM customers;

-- 高カーディナリティ列へのインデックス（効果的）
CREATE INDEX idx_customers_email ON customers(email);

-- 低カーディナリティ列へのインデックス（効果は限定的）
CREATE INDEX idx_customers_gender ON customers(gender);  -- 性別など
```

### 複合インデックスの列順序

```sql
-- 検索パターンに応じた複合インデックス
-- よく使うクエリ: WHERE city = ? AND age > ?
CREATE INDEX idx_city_age ON customers(city, age);

-- このインデックスは以下のクエリで有効
EXPLAIN QUERY PLAN SELECT * FROM customers WHERE city = '東京';
EXPLAIN QUERY PLAN SELECT * FROM customers WHERE city = '東京' AND age > 30;

-- しかし、以下のクエリでは効果が限定的
EXPLAIN QUERY PLAN SELECT * FROM customers WHERE age > 30;  -- 最初の列を使わない

-- 両方のパターンに対応する場合
CREATE INDEX idx_age ON customers(age);
```

### カバリングインデックス

```sql
-- クエリに必要なすべての列を含むインデックス
CREATE INDEX idx_covering ON customers(city, age, name);

-- このクエリはインデックスのみで完結（テーブルアクセス不要）
EXPLAIN QUERY PLAN 
SELECT city, age, name FROM customers WHERE city = '東京' AND age > 30;

-- パフォーマンス比較
.timer on
-- 通常のインデックス
SELECT city, age, name, email FROM customers WHERE city = '東京' AND age > 30;
-- カバリングインデックス
SELECT city, age, name FROM customers WHERE city = '東京' AND age > 30;
.timer off
```

## 6.5 実行計画の分析

### EXPLAIN QUERY PLAN

```sql
-- 基本的な実行計画
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE age = 25;

-- JOINを含む実行計画
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER,
    order_date TEXT,
    total_amount REAL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE INDEX idx_orders_customer ON orders(customer_id);

EXPLAIN QUERY PLAN
SELECT c.name, COUNT(o.id) as order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.city = '東京'
GROUP BY c.id, c.name;
```

### インデックスの使用状況確認

```sql
-- インデックス一覧
SELECT 
    name,
    tbl_name,
    sql
FROM sqlite_master 
WHERE type = 'index' AND tbl_name = 'customers';

-- インデックスの詳細情報
PRAGMA index_list(customers);
PRAGMA index_info(idx_customers_city);

-- インデックスの使用統計（分析用）
ANALYZE;
SELECT * FROM sqlite_stat1 WHERE tbl = 'customers';
```

## 6.6 クエリの最適化

### インデックスが使われない場合

```sql
-- 関数や演算を使用した場合
-- 悪い例：インデックスが使われない
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE UPPER(city) = 'TOKYO';

-- 良い例：インデックスが使われる
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE city = '東京';

-- 式インデックスで対応
CREATE INDEX idx_customers_city_upper ON customers(UPPER(city));

-- LIKE演算子の使用
-- 前方一致はインデックスが使われる
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE email LIKE 'customer1%';

-- 中間一致や後方一致は使われない
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE email LIKE '%@example.com';
```

### OR条件の最適化

```sql
-- OR条件は各条件で別々にインデックスが必要
CREATE INDEX idx_customers_age ON customers(age);
CREATE INDEX idx_customers_city ON customers(city);

-- 非効率なクエリ
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE age = 25 OR city = '東京';

-- UNIONを使った最適化
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE age = 25
UNION
SELECT * FROM customers WHERE city = '東京';

-- INを使った最適化（同じ列の場合）
EXPLAIN QUERY PLAN
SELECT * FROM customers WHERE city IN ('東京', '大阪', '名古屋');
```

### JOINの最適化

```sql
-- 適切なインデックスがある場合
CREATE INDEX idx_orders_date ON orders(order_date);

EXPLAIN QUERY PLAN
SELECT c.name, o.order_date, o.total_amount
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
WHERE o.order_date >= '2024-01-01';

-- 複合インデックスでさらに最適化
CREATE INDEX idx_orders_date_customer ON orders(order_date, customer_id);

-- 小さいテーブルを先に絞り込む
EXPLAIN QUERY PLAN
SELECT c.name, o.order_date, o.total_amount
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
WHERE o.order_date >= '2024-01-01' AND o.total_amount > 10000;
```

## 6.7 パフォーマンスチューニング

### データベース全体の最適化

```sql
-- 統計情報の更新
ANALYZE;

-- データベースの最適化（断片化の解消）
VACUUM;

-- 自動VACUUM設定
PRAGMA auto_vacuum = FULL;

-- ページサイズの確認と設定（新規DBのみ）
PRAGMA page_size;
-- PRAGMA page_size = 4096;

-- キャッシュサイズの調整
PRAGMA cache_size = -64000;  -- 64MBのキャッシュ

-- WALモードの有効化（並行性の向上）
PRAGMA journal_mode = WAL;
```

### メモリ使用の最適化

```sql
-- 一時ストレージをメモリに
PRAGMA temp_store = MEMORY;

-- メモリマップI/Oの使用
PRAGMA mmap_size = 268435456;  -- 256MB

-- 同期モードの調整（パフォーマンス優先）
PRAGMA synchronous = NORMAL;  -- デフォルトはFULL
```

### バッチ処理の最適化

```sql
-- トランザクションを使用したバッチ挿入
BEGIN TRANSACTION;
-- 大量のINSERT文
INSERT INTO customers (name, email, age, city) VALUES (...);
-- ... 数千件のINSERT
COMMIT;

-- プリペアドステートメントの使用（プログラミング言語から）
-- Pythonの例
/*
import sqlite3
conn = sqlite3.connect('database.db')
cursor = conn.cursor()
cursor.executemany(
    "INSERT INTO customers (name, email, age, city) VALUES (?, ?, ?, ?)",
    data_list
)
conn.commit()
*/
```

## 6.8 インデックスの管理

### インデックスの再構築

```sql
-- インデックスの削除と再作成
DROP INDEX IF EXISTS idx_customers_city;
CREATE INDEX idx_customers_city ON customers(city);

-- REINDEXコマンド
REINDEX idx_customers_city;
REINDEX customers;  -- テーブルのすべてのインデックス

-- データベース全体の再インデックス
REINDEX;
```

### 不要なインデックスの特定

```sql
-- 使用されていないインデックスの候補を見つける
-- （実際の使用状況はアプリケーションログから判断）
SELECT 
    i.name AS index_name,
    m.tbl_name AS table_name,
    i.partial,
    GROUP_CONCAT(ii.name) AS columns
FROM sqlite_master m
JOIN pragma_index_list(m.tbl_name) i
JOIN pragma_index_info(i.name) ii
WHERE m.type = 'table'
GROUP BY i.name, m.tbl_name, i.partial
ORDER BY m.tbl_name, i.name;
```

### インデックスのコスト

```sql
-- インデックスのサイズ確認
SELECT 
    name,
    tbl_name,
    ROUND(pgsize / 1024.0, 2) AS size_kb
FROM (
    SELECT 
        name,
        tbl_name,
        (SELECT SUM(pgsize) FROM dbstat WHERE name = idx.name) AS pgsize
    FROM sqlite_master idx
    WHERE type = 'index'
)
ORDER BY pgsize DESC;

-- 更新パフォーマンスへの影響測定
.timer on
-- インデックスが多い場合の更新
UPDATE customers SET age = age + 1 WHERE id < 1000;
.timer off
```

## 6.9 実践的なパフォーマンス改善例

### ケース1: 検索クエリの最適化

```sql
-- 問題のあるクエリ
-- 実行時間: 数秒
SELECT 
    c.name,
    COUNT(o.id) AS order_count,
    SUM(o.total_amount) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.created_at >= '2024-01-01'
    AND c.city IN ('東京', '大阪')
    AND (o.order_date >= '2024-01-01' OR o.order_date IS NULL)
GROUP BY c.id, c.name
HAVING COUNT(o.id) > 0
ORDER BY total_spent DESC
LIMIT 100;

-- ステップ1: 実行計画の確認
EXPLAIN QUERY PLAN [上記クエリ];

-- ステップ2: 必要なインデックスの作成
CREATE INDEX idx_customers_created_city ON customers(created_at, city);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- ステップ3: クエリの書き換え
WITH customer_subset AS (
    SELECT id, name
    FROM customers
    WHERE created_at >= '2024-01-01'
        AND city IN ('東京', '大阪')
),
order_summary AS (
    SELECT 
        customer_id,
        COUNT(*) AS order_count,
        SUM(total_amount) AS total_spent
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY customer_id
)
SELECT 
    cs.name,
    COALESCE(os.order_count, 0) AS order_count,
    COALESCE(os.total_spent, 0) AS total_spent
FROM customer_subset cs
INNER JOIN order_summary os ON cs.id = os.customer_id
ORDER BY os.total_spent DESC
LIMIT 100;
```

### ケース2: 集計クエリの最適化

```sql
-- 日次売上集計の高速化
-- 元のクエリ
SELECT 
    DATE(order_date) AS date,
    COUNT(*) AS order_count,
    SUM(total_amount) AS daily_total
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY DATE(order_date)
ORDER BY date;

-- 最適化版
-- 1. 式インデックスの作成
CREATE INDEX idx_orders_date_only ON orders(DATE(order_date));

-- 2. マテリアライズドビューの代替
CREATE TABLE daily_sales_cache AS
SELECT 
    DATE(order_date) AS date,
    COUNT(*) AS order_count,
    SUM(total_amount) AS daily_total
FROM orders
GROUP BY DATE(order_date);

CREATE INDEX idx_daily_sales_date ON daily_sales_cache(date);

-- 3. トリガーで自動更新
CREATE TRIGGER update_daily_sales
AFTER INSERT ON orders
BEGIN
    INSERT OR REPLACE INTO daily_sales_cache (date, order_count, daily_total)
    SELECT 
        DATE(NEW.order_date),
        COALESCE((SELECT order_count FROM daily_sales_cache WHERE date = DATE(NEW.order_date)), 0) + 1,
        COALESCE((SELECT daily_total FROM daily_sales_cache WHERE date = DATE(NEW.order_date)), 0) + NEW.total_amount;
END;
```

## 6.10 パフォーマンス監視

### クエリプロファイリング

```sql
-- 実行時間の詳細な測定
.timer on
.echo on

-- スロークエリの特定
CREATE TABLE query_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    query TEXT,
    execution_time REAL,
    timestamp TEXT DEFAULT CURRENT_TIMESTAMP
);

-- アプリケーション側でクエリ実行時間を記録
-- 閾値を超えたクエリをログに記録
```

### システムヘルスチェック

```sql
-- データベースサイズの確認
SELECT 
    page_count * page_size / 1024 / 1024 AS size_mb
FROM pragma_page_count(), pragma_page_size();

-- フラグメンテーションの確認
PRAGMA freelist_count;

-- インテグリティチェック
PRAGMA integrity_check;

-- 最適化の提案
PRAGMA optimize;
```

## 6.11 ベストプラクティス

### インデックス設計のガイドライン

1. **WHERE句で頻繁に使用される列にインデックスを作成**
2. **JOINの結合キーにインデックスを作成**
3. **ORDER BY、GROUP BYで使用される列を考慮**
4. **複合インデックスは最も選択的な列を先頭に**
5. **更新頻度とのバランスを考慮**

### アンチパターンの回避

```sql
-- 避けるべき: 過度なインデックス
-- すべての列にインデックスを作成しない

-- 避けるべき: 重複したインデックス
CREATE INDEX idx1 ON table(col1);
CREATE INDEX idx2 ON table(col1, col2);  -- idx1は不要

-- 避けるべき: 低選択性の列への単独インデックス
CREATE INDEX idx_gender ON users(gender);  -- 効果が限定的

-- 避けるべき: 頻繁に更新される列への過度なインデックス
CREATE INDEX idx_last_updated ON users(last_updated);
```

## 6.12 まとめ

この章では、SQLiteのインデックスとパフォーマンス最適化について学びました。適切なインデックス設計とクエリの最適化により、データベースのパフォーマンスを大幅に改善できます。

### この章で学んだこと

- インデックスの基本概念とB-Tree構造
- 各種インデックスの作成方法と使い分け
- 実行計画の読み方と分析方法
- クエリ最適化のテクニック
- データベース全体のパフォーマンスチューニング
- インデックスの管理とメンテナンス

### 確認問題

1. カーディナリティが低い列にインデックスを作成することの問題点を説明してください
2. カバリングインデックスの利点と、それが有効な場面を説明してください
3. EXPLAIN QUERY PLANの出力を見て、クエリがインデックスを使用しているか判断する方法を説明してください
4. 複合インデックス(col1, col2, col3)がある場合、どのようなWHERE句で有効に使われるか例を挙げてください

---

**[← 第5章: 高度なクエリへ戻る](./chapter05-advanced-queries.md)** | **[→ 第7章: トランザクションへ進む](./chapter07-transactions.md)**