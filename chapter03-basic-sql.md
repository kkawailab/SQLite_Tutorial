# 第3章: 基本的なSQL操作

## 3.1 はじめに

この章では、SQLiteで最も基本的かつ重要なSQL操作を学びます。データベースの作成から始まり、テーブルの作成、データの挿入、検索、更新、削除といった基本的な操作（CRUD: Create, Read, Update, Delete）を順を追って解説します。

## 3.2 データベースの作成と接続

### データベースファイルの作成

SQLiteでは、データベースは単一のファイルとして作成されます。

```bash
# 新しいデータベースの作成
sqlite3 myshop.db

# 既存のデータベースに接続
sqlite3 existing_database.db

# 一時的なメモリデータベース
sqlite3 :memory:
```

### データベースの基本情報確認

```sql
-- 現在接続しているデータベースの確認
.databases

-- SQLiteのバージョン確認
SELECT sqlite_version();

-- 現在の日時確認
SELECT datetime('now', 'localtime');
```

## 3.3 テーブルの作成（CREATE TABLE）

### 基本的なテーブル作成

```sql
-- 商品テーブルの作成
CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    price REAL NOT NULL,
    stock INTEGER DEFAULT 0,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- 顧客テーブルの作成
CREATE TABLE customers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    phone TEXT,
    address TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- 注文テーブルの作成
CREATE TABLE orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    customer_id INTEGER NOT NULL,
    order_date TEXT DEFAULT CURRENT_TIMESTAMP,
    total_amount REAL NOT NULL,
    status TEXT DEFAULT 'pending',
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

### テーブル構造の確認

```sql
-- すべてのテーブルを表示
.tables

-- 特定のテーブルの構造を確認
.schema products

-- より詳細な情報
PRAGMA table_info(products);

-- すべてのテーブルのスキーマを表示
.schema
```

### IF NOT EXISTSを使用した安全なテーブル作成

```sql
-- テーブルが存在しない場合のみ作成
CREATE TABLE IF NOT EXISTS categories (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    description TEXT
);
```

## 3.4 データの挿入（INSERT）

### 基本的なINSERT文

```sql
-- 単一行の挿入
INSERT INTO products (name, price, stock) 
VALUES ('ノートパソコン', 89800, 10);

-- すべての列に値を指定する場合
INSERT INTO products 
VALUES (NULL, 'マウス', 2980, 50, datetime('now', 'localtime'));

-- 複数行の一括挿入
INSERT INTO products (name, price, stock) VALUES 
    ('キーボード', 5980, 30),
    ('モニター', 29800, 15),
    ('USBメモリ 32GB', 1980, 100);
```

### 顧客データの挿入

```sql
-- 顧客データの挿入
INSERT INTO customers (name, email, phone, address) VALUES 
    ('山田太郎', 'yamada@example.com', '090-1234-5678', '東京都渋谷区'),
    ('鈴木花子', 'suzuki@example.com', '080-9876-5432', '大阪府大阪市'),
    ('田中一郎', 'tanaka@example.com', NULL, '愛知県名古屋市');
```

### INSERT OR REPLACEの使用

```sql
-- 重複時は置き換える
INSERT OR REPLACE INTO categories (id, name, description) 
VALUES (1, 'コンピュータ', 'PC本体と周辺機器');

-- INSERT OR IGNOREで重複を無視
INSERT OR IGNORE INTO categories (name, description) 
VALUES ('コンピュータ', '既に存在する場合は無視される');
```

## 3.5 データの検索（SELECT）

### 基本的なSELECT文

```sql
-- すべての列を取得
SELECT * FROM products;

-- 特定の列のみ取得
SELECT name, price FROM products;

-- 列に別名をつける
SELECT 
    name AS 商品名, 
    price AS 価格,
    stock AS 在庫数
FROM products;
```

### WHERE句による条件指定

```sql
-- 価格が10000円以上の商品
SELECT * FROM products WHERE price >= 10000;

-- 在庫が20個以下の商品
SELECT name, stock FROM products WHERE stock <= 20;

-- 複数条件（AND）
SELECT * FROM products 
WHERE price >= 5000 AND stock > 0;

-- 複数条件（OR）
SELECT * FROM products 
WHERE price < 3000 OR stock > 50;

-- 文字列の部分一致（LIKE）
SELECT * FROM customers 
WHERE email LIKE '%@example.com';

-- 名前が「田」で始まる顧客
SELECT * FROM customers 
WHERE name LIKE '田%';
```

### ORDER BYによる並び替え

```sql
-- 価格の安い順
SELECT * FROM products ORDER BY price ASC;

-- 価格の高い順
SELECT * FROM products ORDER BY price DESC;

-- 複数条件での並び替え
SELECT * FROM products 
ORDER BY stock DESC, price ASC;
```

### LIMITによる件数制限

```sql
-- 上位5件のみ取得
SELECT * FROM products 
ORDER BY price DESC 
LIMIT 5;

-- 6件目から5件取得（ページング）
SELECT * FROM products 
ORDER BY id 
LIMIT 5 OFFSET 5;
```

### 集計関数

```sql
-- 商品の総数
SELECT COUNT(*) AS 商品総数 FROM products;

-- 平均価格
SELECT AVG(price) AS 平均価格 FROM products;

-- 最高価格と最低価格
SELECT 
    MAX(price) AS 最高価格,
    MIN(price) AS 最低価格
FROM products;

-- 在庫の合計
SELECT SUM(stock) AS 在庫総数 FROM products;

-- 条件付き集計
SELECT COUNT(*) AS 高額商品数 
FROM products 
WHERE price >= 10000;
```

### GROUP BYによるグループ化

```sql
-- まずカテゴリを商品に追加
ALTER TABLE products ADD COLUMN category_id INTEGER;

-- サンプルデータの更新
UPDATE products SET category_id = 1 WHERE name LIKE '%パソコン%' OR name LIKE '%モニター%';
UPDATE products SET category_id = 2 WHERE name LIKE '%マウス%' OR name LIKE '%キーボード%';

-- カテゴリごとの商品数
SELECT 
    category_id,
    COUNT(*) AS 商品数,
    AVG(price) AS 平均価格
FROM products
GROUP BY category_id;

-- HAVING句での絞り込み
SELECT 
    category_id,
    COUNT(*) AS 商品数
FROM products
GROUP BY category_id
HAVING COUNT(*) >= 2;
```

## 3.6 データの更新（UPDATE）

### 基本的なUPDATE文

```sql
-- 単一レコードの更新
UPDATE products 
SET price = 85000 
WHERE id = 1;

-- 複数の列を更新
UPDATE products 
SET 
    price = price * 0.9,  -- 10%値下げ
    stock = stock + 10
WHERE name = 'ノートパソコン';

-- 条件に合致する複数レコードの更新
UPDATE products 
SET stock = stock + 20 
WHERE stock < 30;
```

### 計算を含む更新

```sql
-- 価格を10%値上げ
UPDATE products 
SET price = price * 1.1 
WHERE category_id = 1;

-- 在庫を半分に
UPDATE products 
SET stock = stock / 2 
WHERE stock > 100;

-- 現在時刻で更新
UPDATE customers 
SET updated_at = datetime('now', 'localtime') 
WHERE id = 1;
```

### JOINを使用した更新

```sql
-- 注文データの挿入
INSERT INTO orders (customer_id, total_amount, status) VALUES
    (1, 92780, 'completed'),
    (2, 35760, 'completed'),
    (3, 7960, 'pending');

-- 注文履歴のある顧客にフラグを立てる（まず列を追加）
ALTER TABLE customers ADD COLUMN has_ordered BOOLEAN DEFAULT 0;

-- 注文のある顧客を更新
UPDATE customers 
SET has_ordered = 1 
WHERE id IN (SELECT DISTINCT customer_id FROM orders);
```

## 3.7 データの削除（DELETE）

### 基本的なDELETE文

```sql
-- 特定のレコードを削除
DELETE FROM products WHERE id = 5;

-- 条件に合致する複数レコードを削除
DELETE FROM products WHERE stock = 0;

-- 価格が1000円未満の商品を削除
DELETE FROM products WHERE price < 1000;
```

### 安全な削除操作

```sql
-- 削除前に確認（SELECT文で対象を確認）
SELECT * FROM products WHERE stock = 0;

-- トランザクションを使用した安全な削除
BEGIN TRANSACTION;
DELETE FROM products WHERE stock = 0;
-- 確認後、問題なければ
COMMIT;
-- 問題があれば
-- ROLLBACK;
```

### テーブルの全データ削除

```sql
-- すべてのレコードを削除（構造は保持）
DELETE FROM products;

-- より高速な全削除（SQLite 3.35.0以降）
-- TRUNCATE TABLE products;  -- SQLiteでは未サポート

-- 代替方法：テーブルを削除して再作成
DROP TABLE IF EXISTS temp_table;
-- その後、CREATE TABLEで再作成
```

## 3.8 実践演習

### 演習1: 商品管理システムの構築

```sql
-- 1. カテゴリマスタの作成と初期データ投入
CREATE TABLE IF NOT EXISTS categories (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);

INSERT INTO categories (name) VALUES 
    ('コンピュータ'),
    ('周辺機器'),
    ('ソフトウェア'),
    ('アクセサリ');

-- 2. 商品データの整備
UPDATE products SET category_id = 
    CASE 
        WHEN name LIKE '%パソコン%' THEN 1
        WHEN name LIKE '%モニター%' THEN 1
        WHEN name LIKE '%マウス%' OR name LIKE '%キーボード%' THEN 2
        WHEN name LIKE '%USB%' THEN 4
        ELSE 3
    END;

-- 3. カテゴリ別売上ランキング
SELECT 
    c.name AS カテゴリ,
    p.name AS 商品名,
    p.price AS 価格,
    p.stock AS 在庫
FROM products p
JOIN categories c ON p.category_id = c.id
ORDER BY c.name, p.price DESC;
```

### 演習2: 在庫管理レポート

```sql
-- 在庫切れ警告リスト
SELECT 
    name AS 商品名,
    stock AS 現在在庫,
    CASE 
        WHEN stock = 0 THEN '在庫切れ'
        WHEN stock < 10 THEN '要発注'
        ELSE '在庫十分'
    END AS 在庫状態
FROM products
WHERE stock < 10
ORDER BY stock ASC;

-- カテゴリ別在庫金額
SELECT 
    c.name AS カテゴリ,
    COUNT(p.id) AS 商品数,
    SUM(p.stock) AS 総在庫数,
    SUM(p.price * p.stock) AS 在庫金額
FROM products p
JOIN categories c ON p.category_id = c.id
GROUP BY c.id
ORDER BY 在庫金額 DESC;
```

## 3.9 ベストプラクティス

### 1. 明示的な列指定

```sql
-- 良い例：列を明示的に指定
INSERT INTO products (name, price, stock) 
VALUES ('新商品', 5000, 20);

-- 避けるべき：列指定なし
INSERT INTO products 
VALUES (NULL, '新商品', 5000, 20, NULL, NULL);
```

### 2. WHERE句の忘れ防止

```sql
-- 更新・削除前は必ずSELECTで確認
SELECT * FROM products WHERE price > 50000;
-- 確認後
UPDATE products SET price = price * 0.95 WHERE price > 50000;
```

### 3. トランザクションの活用

```sql
BEGIN TRANSACTION;
-- 複数の関連操作
INSERT INTO orders (customer_id, total_amount) VALUES (1, 10000);
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

## 3.10 まとめ

この章では、SQLiteにおける基本的なSQL操作を学びました。CREATE TABLE、INSERT、SELECT、UPDATE、DELETEという基本的な命令を使いこなせるようになることで、データベースの基本的な操作が可能になります。

### この章で学んだこと

- データベースとテーブルの作成方法
- データの挿入、検索、更新、削除の基本操作
- WHERE句による条件指定
- ORDER BY、LIMIT、GROUP BYなどの応用的な検索
- 集計関数の使い方
- 安全なデータ操作のベストプラクティス

### 確認問題

1. 価格が5000円以上10000円以下の商品を検索するSQLを書いてください
2. 全商品の価格を15%値上げするUPDATE文を書いてください
3. カテゴリごとの最高価格商品を表示するSQLを書いてください
4. 在庫が0の商品を削除する前に、対象商品を確認するSQLを書いてください

---

**[← 第2章: 環境構築へ戻る](./chapter02-setup.md)** | **[→ 第4章: データ型と制約へ進む](./chapter04-datatypes-constraints.md)**