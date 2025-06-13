# 第4章: データ型と制約

## 4.1 はじめに

この章では、SQLiteのデータ型システムと、データの整合性を保つための制約について詳しく学びます。SQLiteは他のデータベースシステムとは異なる独特な型システムを持っているため、その特徴を理解することが重要です。

## 4.2 SQLiteのデータ型システム

### 動的型付けシステム

SQLiteは「動的型付け」を採用しています。これは、値自体が型を持ち、列は型の「親和性（affinity）」を持つという考え方です。

```sql
-- SQLiteでは、異なる型の値を同じ列に格納できる
CREATE TABLE flexible_table (
    col1 TEXT
);

INSERT INTO flexible_table VALUES 
    ('文字列'),
    (123),      -- 数値も格納可能
    (45.67),    -- 実数も格納可能
    (NULL);     -- NULLも格納可能

SELECT col1, typeof(col1) FROM flexible_table;
```

### ストレージクラス

SQLiteには5つの基本的なストレージクラスがあります：

1. **NULL** - NULL値
2. **INTEGER** - 符号付き整数（1, 2, 3, 4, 6, または 8 バイト）
3. **REAL** - 浮動小数点数（8バイトのIEEE浮動小数点数）
4. **TEXT** - テキスト文字列（UTF-8, UTF-16BE, UTF-16LE）
5. **BLOB** - バイナリデータ（Binary Large Object）

```sql
-- 各ストレージクラスの例
CREATE TABLE storage_examples (
    null_col NULL,
    int_col INTEGER,
    real_col REAL,
    text_col TEXT,
    blob_col BLOB
);

INSERT INTO storage_examples VALUES 
    (NULL, 42, 3.14159, 'Hello, SQLite!', x'48656C6C6F');

-- 型の確認
SELECT 
    typeof(null_col),
    typeof(int_col),
    typeof(real_col),
    typeof(text_col),
    typeof(blob_col)
FROM storage_examples;
```

## 4.3 型親和性（Type Affinity）

### 親和性の種類

SQLiteの列には以下の5つの型親和性があります：

1. **TEXT** - 文字列を優先
2. **NUMERIC** - 数値を優先（整数または実数）
3. **INTEGER** - 整数を優先
4. **REAL** - 実数を優先
5. **BLOB** - そのまま格納（親和性なし）

### 型親和性の決定ルール

```sql
-- 様々な型定義と対応する親和性
CREATE TABLE type_affinity_examples (
    -- INTEGER親和性
    col_int INT,
    col_integer INTEGER,
    col_tinyint TINYINT,
    col_bigint BIGINT,
    
    -- TEXT親和性
    col_char CHAR(10),
    col_varchar VARCHAR(255),
    col_text TEXT,
    col_clob CLOB,
    
    -- REAL親和性
    col_real REAL,
    col_double DOUBLE,
    col_float FLOAT,
    
    -- NUMERIC親和性
    col_numeric NUMERIC,
    col_decimal DECIMAL(10,2),
    col_boolean BOOLEAN,
    col_date DATE,
    
    -- BLOB親和性（親和性なし）
    col_blob BLOB
);
```

### 型変換の例

```sql
CREATE TABLE type_conversion (
    text_col TEXT,
    int_col INTEGER,
    real_col REAL,
    numeric_col NUMERIC
);

-- 同じ値'123'を異なる親和性の列に挿入
INSERT INTO type_conversion VALUES 
    ('123', '123', '123', '123');

-- 格納された型を確認
SELECT 
    text_col, typeof(text_col),
    int_col, typeof(int_col),
    real_col, typeof(real_col),
    numeric_col, typeof(numeric_col)
FROM type_conversion;
```

## 4.4 PRIMARY KEY制約

### 基本的なPRIMARY KEY

```sql
-- 方法1: 列定義で指定
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    email TEXT NOT NULL
);

-- 方法2: テーブル制約として指定
CREATE TABLE products (
    id INTEGER,
    name TEXT NOT NULL,
    price REAL NOT NULL,
    PRIMARY KEY (id)
);

-- 複合PRIMARY KEY
CREATE TABLE order_items (
    order_id INTEGER,
    product_id INTEGER,
    quantity INTEGER NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

### AUTOINCREMENT

```sql
-- AUTOINCREMENTの使用
CREATE TABLE logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    message TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- 通常のINTEGER PRIMARY KEYとの違い
CREATE TABLE comparison1 (
    id INTEGER PRIMARY KEY,
    data TEXT
);

CREATE TABLE comparison2 (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    data TEXT
);

-- rowidの再利用の違いを確認
INSERT INTO comparison1 (data) VALUES ('test1'), ('test2');
DELETE FROM comparison1 WHERE id = 2;
INSERT INTO comparison1 (data) VALUES ('test3');
SELECT * FROM comparison1;

INSERT INTO comparison2 (data) VALUES ('test1'), ('test2');
DELETE FROM comparison2 WHERE id = 2;
INSERT INTO comparison2 (data) VALUES ('test3');
SELECT * FROM comparison2;
```

## 4.5 FOREIGN KEY制約

### 外部キーの有効化

```sql
-- 外部キー制約はデフォルトで無効なので有効化が必要
PRAGMA foreign_keys = ON;

-- 現在の設定確認
PRAGMA foreign_keys;
```

### 基本的な外部キー

```sql
-- 親テーブル
CREATE TABLE departments (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL
);

-- 子テーブル（外部キー制約付き）
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    department_id INTEGER,
    FOREIGN KEY (department_id) REFERENCES departments(id)
);

-- データの挿入
INSERT INTO departments (name) VALUES ('営業部'), ('開発部'), ('総務部');

-- 正常な挿入
INSERT INTO employees (name, department_id) VALUES ('山田太郎', 1);

-- エラーになる挿入（存在しない部署）
-- INSERT INTO employees (name, department_id) VALUES ('鈴木花子', 99);
```

### カスケード操作

```sql
-- ON DELETE CASCADE と ON UPDATE CASCADE
CREATE TABLE categories (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE items (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    category_id INTEGER,
    FOREIGN KEY (category_id) REFERENCES categories(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);

-- テストデータ
INSERT INTO categories (id, name) VALUES (1, '家電');
INSERT INTO items (name, category_id) VALUES 
    ('テレビ', 1),
    ('冷蔵庫', 1);

-- カテゴリを削除すると関連アイテムも削除される
DELETE FROM categories WHERE id = 1;
SELECT * FROM items; -- 空になる
```

### その他のカスケードオプション

```sql
-- 様々なカスケードオプション
CREATE TABLE parent_table (
    id INTEGER PRIMARY KEY,
    name TEXT
);

CREATE TABLE child_table (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER,
    name TEXT,
    -- SET NULL: 親が削除されたらNULLに設定
    FOREIGN KEY (parent_id) REFERENCES parent_table(id)
        ON DELETE SET NULL
);

CREATE TABLE child_table2 (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER DEFAULT 0,
    name TEXT,
    -- SET DEFAULT: 親が削除されたらデフォルト値に設定
    FOREIGN KEY (parent_id) REFERENCES parent_table(id)
        ON DELETE SET DEFAULT
);

CREATE TABLE child_table3 (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER,
    name TEXT,
    -- RESTRICT: 子レコードがある場合は親の削除を禁止
    FOREIGN KEY (parent_id) REFERENCES parent_table(id)
        ON DELETE RESTRICT
);
```

## 4.6 その他の制約

### NOT NULL制約

```sql
CREATE TABLE required_fields (
    id INTEGER PRIMARY KEY,
    -- NOT NULL制約
    required_text TEXT NOT NULL,
    optional_text TEXT,
    -- デフォルト値付きNOT NULL
    status TEXT NOT NULL DEFAULT 'active',
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- NOT NULLフィールドへのNULL挿入はエラー
-- INSERT INTO required_fields (required_text) VALUES (NULL);

-- 正常な挿入
INSERT INTO required_fields (required_text) VALUES ('必須データ');
```

### UNIQUE制約

```sql
CREATE TABLE unique_examples (
    id INTEGER PRIMARY KEY,
    -- 単一列のUNIQUE制約
    email TEXT UNIQUE,
    -- 別の書き方
    username TEXT,
    UNIQUE(username)
);

-- 複合UNIQUE制約
CREATE TABLE composite_unique (
    id INTEGER PRIMARY KEY,
    category TEXT,
    code TEXT,
    name TEXT,
    -- categoryとcodeの組み合わせがユニーク
    UNIQUE(category, code)
);

-- テスト
INSERT INTO composite_unique (category, code, name) VALUES 
    ('A', '001', '商品1'),
    ('A', '002', '商品2'),
    ('B', '001', '商品3'); -- OK: カテゴリが違えば同じコードでも可

-- エラーになる
-- INSERT INTO composite_unique (category, code, name) VALUES ('A', '001', '商品4');
```

### CHECK制約

```sql
CREATE TABLE check_constraints (
    id INTEGER PRIMARY KEY,
    age INTEGER CHECK(age >= 0 AND age <= 150),
    price REAL CHECK(price > 0),
    discount_rate REAL CHECK(discount_rate >= 0 AND discount_rate <= 1),
    status TEXT CHECK(status IN ('active', 'inactive', 'pending')),
    start_date TEXT,
    end_date TEXT,
    CHECK(start_date < end_date)
);

-- 正常な挿入
INSERT INTO check_constraints (age, price, discount_rate, status, start_date, end_date) 
VALUES (25, 100.0, 0.1, 'active', '2024-01-01', '2024-12-31');

-- CHECK制約違反の例
-- INSERT INTO check_constraints (age) VALUES (-5); -- 年齢が負
-- INSERT INTO check_constraints (price) VALUES (0); -- 価格が0以下
-- INSERT INTO check_constraints (status) VALUES ('unknown'); -- 無効なステータス
```

### DEFAULT制約

```sql
CREATE TABLE default_values (
    id INTEGER PRIMARY KEY,
    -- リテラル値のデフォルト
    status TEXT DEFAULT 'active',
    quantity INTEGER DEFAULT 0,
    is_visible BOOLEAN DEFAULT 1,
    
    -- 関数を使用したデフォルト
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT DEFAULT CURRENT_TIMESTAMP,
    
    -- 式を使用したデフォルト
    code TEXT DEFAULT (lower(hex(randomblob(4)))),
    
    -- NULLをデフォルトに（明示的）
    description TEXT DEFAULT NULL
);

-- デフォルト値の確認
INSERT INTO default_values (id) VALUES (1);
SELECT * FROM default_values;
```

## 4.7 制約の管理

### 制約の確認

```sql
-- テーブルの制約情報を確認
PRAGMA table_info(check_constraints);

-- 外部キー情報の確認
PRAGMA foreign_key_list(employees);

-- インデックス情報の確認（PRIMARY KEYとUNIQUE制約）
PRAGMA index_list(unique_examples);
```

### 制約の追加と削除

```sql
-- SQLiteでは既存テーブルへの制約追加は限定的
-- 新しい列の追加時に制約を指定
ALTER TABLE existing_table 
ADD COLUMN new_column TEXT NOT NULL DEFAULT 'default_value';

-- 制約を変更する場合は、通常テーブルの再作成が必要
-- 1. 新しいテーブルを作成
CREATE TABLE new_table (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE
);

-- 2. データをコピー
INSERT INTO new_table SELECT * FROM old_table;

-- 3. 古いテーブルを削除
DROP TABLE old_table;

-- 4. 新しいテーブルをリネーム
ALTER TABLE new_table RENAME TO old_table;
```

## 4.8 実践的な例

### ユーザー管理システム

```sql
-- 実践的なユーザー管理テーブル
CREATE TABLE users_advanced (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL UNIQUE,
    email TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    age INTEGER CHECK(age >= 13),  -- 13歳以上
    status TEXT NOT NULL DEFAULT 'active' CHECK(status IN ('active', 'inactive', 'suspended')),
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_login_at TEXT,
    CHECK(email LIKE '%@%.%')  -- 簡易的なメールフォーマットチェック
);

-- ロールテーブル
CREATE TABLE roles (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    description TEXT
);

-- ユーザーとロールの関連テーブル
CREATE TABLE user_roles (
    user_id INTEGER NOT NULL,
    role_id INTEGER NOT NULL,
    assigned_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users_advanced(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
);
```

### 在庫管理システム

```sql
-- 商品マスタ
CREATE TABLE products_master (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sku TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    category_id INTEGER NOT NULL,
    unit_price REAL NOT NULL CHECK(unit_price > 0),
    min_stock INTEGER NOT NULL DEFAULT 0 CHECK(min_stock >= 0),
    max_stock INTEGER NOT NULL DEFAULT 1000 CHECK(max_stock > min_stock),
    is_active BOOLEAN NOT NULL DEFAULT 1,
    FOREIGN KEY (category_id) REFERENCES categories(id)
);

-- 在庫トランザクション
CREATE TABLE inventory_transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id INTEGER NOT NULL,
    transaction_type TEXT NOT NULL CHECK(transaction_type IN ('in', 'out', 'adjustment')),
    quantity INTEGER NOT NULL,
    balance_after INTEGER NOT NULL CHECK(balance_after >= 0),
    reference_no TEXT,
    notes TEXT,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    FOREIGN KEY (product_id) REFERENCES products_master(id),
    FOREIGN KEY (created_by) REFERENCES users_advanced(id)
);
```

## 4.9 ベストプラクティス

### 1. 適切な型の選択

```sql
-- 良い例：用途に応じた適切な型
CREATE TABLE good_types (
    id INTEGER PRIMARY KEY,
    price REAL,              -- 価格は小数点を含む可能性
    quantity INTEGER,        -- 数量は整数
    description TEXT,        -- 長さが不定のテキスト
    is_active BOOLEAN,       -- 真偽値（実際はINTEGERとして格納）
    created_at TEXT          -- 日時（ISO8601形式の文字列）
);

-- 避けるべき例：不適切な型
CREATE TABLE bad_types (
    id TEXT PRIMARY KEY,     -- IDは数値型が効率的
    price TEXT,              -- 価格を文字列で持つと計算が困難
    quantity REAL            -- 数量に小数は不要
);
```

### 2. 制約の積極的な活用

```sql
-- 制約を活用したテーブル設計
CREATE TABLE well_constrained (
    id INTEGER PRIMARY KEY,
    code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    email TEXT UNIQUE CHECK(email LIKE '%@%.%'),
    age INTEGER CHECK(age > 0 AND age < 150),
    status TEXT NOT NULL DEFAULT 'pending' 
        CHECK(status IN ('pending', 'active', 'inactive')),
    parent_id INTEGER,
    FOREIGN KEY (parent_id) REFERENCES well_constrained(id)
);
```

### 3. NULL値の適切な使用

```sql
-- NULLを許可する場合と許可しない場合の判断
CREATE TABLE null_handling (
    -- 必須項目：NOT NULL
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    
    -- オプション項目：NULL許可
    middle_name TEXT,  -- ミドルネームは持たない人もいる
    phone_number TEXT, -- 電話番号は任意
    
    -- デフォルト値で対応
    status TEXT NOT NULL DEFAULT 'active',
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## 4.10 まとめ

この章では、SQLiteのデータ型システムと各種制約について学びました。SQLiteの動的型付けシステムは柔軟性が高い反面、適切に使用しないとデータの整合性が損なわれる可能性があります。制約を適切に使用することで、データベースレベルでデータの品質を保証できます。

### この章で学んだこと

- SQLiteの5つのストレージクラス（NULL, INTEGER, REAL, TEXT, BLOB）
- 型親和性の概念と動作
- PRIMARY KEY制約とAUTOINCREMENT
- FOREIGN KEY制約とカスケード操作
- NOT NULL、UNIQUE、CHECK、DEFAULT制約
- 実践的なテーブル設計の例

### 確認問題

1. SQLiteの型親和性について説明し、TEXTとNUMERICの違いを述べてください
2. FOREIGN KEY制約のON DELETE CASCADEとON DELETE SET NULLの違いを説明してください
3. CHECK制約を使用して、価格が0より大きく、かつ割引率が0以上1以下であることを保証するテーブルを作成してください
4. INTEGER PRIMARY KEYとINTEGER PRIMARY KEY AUTOINCREMENTの違いを説明してください

---

**[← 第3章: 基本的なSQL操作へ戻る](./chapter03-basic-sql.md)** | **[→ 第5章: 高度なクエリへ進む](./chapter05-advanced-queries.md)**