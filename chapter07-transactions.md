# 第7章: トランザクション

## 7.1 はじめに

この章では、SQLiteにおけるトランザクションの概念と使い方を学びます。トランザクションは、データベースの整合性を保ちながら、複数の操作をまとめて実行するための重要な機能です。ACID特性、同時実行制御、ロックの仕組みなどを理解することで、信頼性の高いデータベースアプリケーションを構築できます。

## 7.2 トランザクションの基本概念

### トランザクションとは

トランザクションは、一連のデータベース操作を1つの論理的な作業単位として扱う仕組みです。

```sql
-- サンプルテーブルの作成
CREATE TABLE accounts (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    balance REAL NOT NULL CHECK(balance >= 0)
);

CREATE TABLE transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    from_account_id INTEGER,
    to_account_id INTEGER,
    amount REAL NOT NULL CHECK(amount > 0),
    transaction_date TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (from_account_id) REFERENCES accounts(id),
    FOREIGN KEY (to_account_id) REFERENCES accounts(id)
);

-- テストデータの挿入
INSERT INTO accounts (id, name, balance) VALUES
    (1, '山田太郎', 100000),
    (2, '鈴木花子', 50000),
    (3, '田中一郎', 75000);
```

### 基本的なトランザクション操作

```sql
-- トランザクションの開始
BEGIN TRANSACTION;
-- または単に BEGIN

-- 送金処理の例
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;
INSERT INTO transactions (from_account_id, to_account_id, amount) 
VALUES (1, 2, 10000);

-- トランザクションのコミット（確定）
COMMIT;

-- 残高確認
SELECT * FROM accounts;
```

### ロールバック

```sql
-- トランザクション開始
BEGIN;

-- 誤った操作
UPDATE accounts SET balance = balance - 1000000 WHERE id = 1;

-- 残高確認（マイナスになっている）
SELECT * FROM accounts WHERE id = 1;

-- 操作を取り消す
ROLLBACK;

-- 残高確認（元に戻っている）
SELECT * FROM accounts WHERE id = 1;
```

## 7.3 ACID特性

### Atomicity（原子性）

トランザクション内のすべての操作は、全て成功するか全て失敗するかのどちらかです。

```sql
-- 原子性のデモンストレーション
BEGIN;

UPDATE accounts SET balance = balance - 20000 WHERE id = 1;
UPDATE accounts SET balance = balance + 20000 WHERE id = 2;

-- 制約違反を意図的に発生させる
UPDATE accounts SET balance = balance - 100000 WHERE id = 1;  -- CHECK制約違反

-- エラーが発生したため、すべての変更がロールバックされる
ROLLBACK;

-- 確認：何も変更されていない
SELECT * FROM accounts;
```

### Consistency（一貫性）

トランザクションの前後で、データベースは一貫した状態を保ちます。

```sql
-- 一貫性を保つトリガーの作成
CREATE TRIGGER check_balance_before_transfer
BEFORE UPDATE ON accounts
FOR EACH ROW
WHEN NEW.balance < 0
BEGIN
    SELECT RAISE(ABORT, '残高不足: 送金できません');
END;

-- トランザクションでテスト
BEGIN;
UPDATE accounts SET balance = balance - 200000 WHERE id = 2;  -- エラー発生
ROLLBACK;
```

### Isolation（独立性）

並行して実行されるトランザクションは、互いに干渉しません。

```sql
-- 分離レベルの確認と設定
PRAGMA read_uncommitted;  -- 0 = OFF (デフォルト), 1 = ON

-- セッション1
BEGIN;
UPDATE accounts SET balance = balance + 5000 WHERE id = 1;
-- COMMITする前に...

-- セッション2（別の接続）
SELECT * FROM accounts WHERE id = 1;  -- 変更前の値が見える
```

### Durability（永続性）

コミットされたトランザクションの結果は、システム障害が発生しても失われません。

```sql
-- 同期モードの設定
PRAGMA synchronous;  -- 確認
PRAGMA synchronous = FULL;  -- 最も安全（デフォルト）

-- ジャーナルモードの確認
PRAGMA journal_mode;
```

## 7.4 トランザクションの種類

### 明示的トランザクション

```sql
-- 明示的に開始と終了を指定
BEGIN TRANSACTION;
-- 複数の操作
INSERT INTO accounts (name, balance) VALUES ('新規顧客', 0);
UPDATE accounts SET balance = balance + 10000 WHERE name = '新規顧客';
COMMIT TRANSACTION;
```

### 暗黙的トランザクション

```sql
-- 単一のSQL文は自動的にトランザクションとして実行される
UPDATE accounts SET balance = balance * 1.01;  -- 全員の残高を1%増加

-- 複数の文を1つのトランザクションにまとめる利点
BEGIN;
UPDATE accounts SET balance = balance * 1.01;
INSERT INTO transactions (from_account_id, to_account_id, amount) 
SELECT NULL, id, balance * 0.01 FROM accounts;
COMMIT;
```

### SAVEPOINT

```sql
-- セーブポイントを使った部分的なロールバック
BEGIN;

UPDATE accounts SET balance = balance - 5000 WHERE id = 1;
SAVEPOINT sp1;

UPDATE accounts SET balance = balance + 5000 WHERE id = 2;
SAVEPOINT sp2;

UPDATE accounts SET balance = balance - 3000 WHERE id = 3;
-- この操作だけ取り消したい

ROLLBACK TO sp2;  -- sp2まで戻る
COMMIT;  -- sp1までの変更は確定される

SELECT * FROM accounts;
```

## 7.5 同時実行制御

### ロックの種類

```sql
-- 共有ロック（読み取りロック）のデモ
-- セッション1
BEGIN;
SELECT * FROM accounts;  -- 共有ロックを取得
-- 他のセッションも読み取り可能

-- 排他ロック（書き込みロック）のデモ
-- セッション1
BEGIN;
UPDATE accounts SET balance = balance + 1000 WHERE id = 1;  -- 排他ロック
-- 他のセッションは待機状態になる
```

### デッドロックの回避

```sql
-- デッドロックが発生しやすいパターン
-- セッション1
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
-- ここで待機...
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;

-- セッション2（同時実行）
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 2;
-- ここで待機...
UPDATE accounts SET balance = balance + 1000 WHERE id = 1;
COMMIT;

-- 解決策：一貫した順序でロックを取得
-- 常にIDの小さい順にUPDATE
```

### タイムアウトの設定

```sql
-- ロックタイムアウトの設定（ミリ秒）
PRAGMA busy_timeout = 5000;  -- 5秒

-- 即座に失敗させる
PRAGMA busy_timeout = 0;

-- デフォルトに戻す
PRAGMA busy_timeout = 0;  -- デフォルト
```

## 7.6 ジャーナルモード

### ロールバックジャーナル（デフォルト）

```sql
-- ジャーナルモードの確認
PRAGMA journal_mode;

-- ロールバックジャーナルモード
PRAGMA journal_mode = DELETE;  -- デフォルト

-- トランザクション中のジャーナルファイル
BEGIN;
UPDATE accounts SET balance = balance * 1.1;
-- この時点で.db-journalファイルが作成される
ROLLBACK;
-- ジャーナルファイルが削除される
```

### WAL（Write-Ahead Logging）モード

```sql
-- WALモードに変更
PRAGMA journal_mode = WAL;

-- WALモードの利点
-- 1. 読み取りと書き込みが同時実行可能
-- 2. より高速な書き込み
-- 3. クラッシュリカバリが改善

-- WALチェックポイント
PRAGMA wal_checkpoint;  -- 手動チェックポイント
PRAGMA wal_autocheckpoint = 1000;  -- 自動チェックポイントの設定

-- WALファイルのサイズ確認
PRAGMA wal_checkpoint(TRUNCATE);  -- WALファイルをリセット
```

## 7.7 実践的なトランザクション設計

### バッチ処理の最適化

```sql
-- 非効率な方法（個別のトランザクション）
-- 各INSERTが個別のトランザクション
INSERT INTO accounts (name, balance) VALUES ('顧客1', 10000);
INSERT INTO accounts (name, balance) VALUES ('顧客2', 20000);
-- ... 1000件

-- 効率的な方法（1つのトランザクション）
BEGIN;
INSERT INTO accounts (name, balance) VALUES ('顧客1', 10000);
INSERT INTO accounts (name, balance) VALUES ('顧客2', 20000);
-- ... 1000件
COMMIT;

-- さらに高速化
PRAGMA synchronous = OFF;  -- 一時的に無効化（注意：リスクあり）
PRAGMA journal_mode = MEMORY;
BEGIN;
-- 大量のINSERT
COMMIT;
PRAGMA synchronous = FULL;  -- 元に戻す
PRAGMA journal_mode = DELETE;
```

### エラーハンドリング

```sql
-- トランザクション内でのエラー処理
DROP TABLE IF EXISTS audit_log;
CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    operation TEXT,
    status TEXT,
    error_message TEXT,
    timestamp TEXT DEFAULT CURRENT_TIMESTAMP
);

-- エラーが発生してもログは残す
BEGIN;
INSERT INTO audit_log (operation, status) VALUES ('送金開始', 'STARTED');

-- エラーが発生する可能性のある操作
UPDATE accounts SET balance = balance - 1000000 WHERE id = 1;

-- エラー時の処理（アプリケーション側で実装）
-- IF error THEN
--     INSERT INTO audit_log (operation, status, error_message) 
--     VALUES ('送金失敗', 'FAILED', 'Insufficient balance');
--     ROLLBACK;
-- ELSE
--     INSERT INTO audit_log (operation, status) VALUES ('送金完了', 'SUCCESS');
--     COMMIT;
-- END IF;
```

### 長時間トランザクションの回避

```sql
-- 問題のある長時間トランザクション
BEGIN;
SELECT * FROM large_table;  -- 大量データの読み込み
-- 長時間の処理...
UPDATE accounts SET balance = balance + 100;
COMMIT;  -- この間、他の書き込みがブロックされる

-- 改善版：必要な部分だけトランザクション化
SELECT * FROM large_table;  -- トランザクション外で読み込み
-- 処理...
BEGIN;
UPDATE accounts SET balance = balance + 100;
COMMIT;  -- 短時間で完了
```

## 7.8 トランザクションとトリガー

### トリガー内でのトランザクション

```sql
-- 監査ログトリガー
CREATE TABLE account_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    account_id INTEGER,
    old_balance REAL,
    new_balance REAL,
    changed_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TRIGGER log_balance_change
AFTER UPDATE OF balance ON accounts
BEGIN
    INSERT INTO account_history (account_id, old_balance, new_balance)
    VALUES (NEW.id, OLD.balance, NEW.balance);
END;

-- トランザクション内でのトリガー動作
BEGIN;
UPDATE accounts SET balance = balance + 5000 WHERE id = 1;
UPDATE accounts SET balance = balance - 5000 WHERE id = 2;
-- トリガーも同じトランザクション内で実行される
ROLLBACK;  -- トリガーの変更もロールバックされる

SELECT * FROM account_history;  -- 空のまま
```

### カスケードトリガー

```sql
-- 連鎖的なトリガーの設定
PRAGMA recursive_triggers = ON;

CREATE TRIGGER update_related_accounts
AFTER UPDATE ON accounts
WHEN NEW.balance < 1000
BEGIN
    UPDATE accounts 
    SET balance = balance + 1000 
    WHERE id = (SELECT id FROM accounts WHERE name = '予備口座');
END;
```

## 7.9 分散トランザクションの考慮

### 複数データベース間の整合性

```sql
-- データベース1への接続
ATTACH DATABASE 'database2.db' AS db2;

-- 複数データベースにまたがるトランザクション
BEGIN;
UPDATE main.accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE db2.accounts SET balance = balance + 1000 WHERE id = 1;
COMMIT;

-- 注意：完全な分散トランザクションではない
-- 両方のデータベースが同じSQLite接続で管理される必要がある
```

### 2フェーズコミットの模擬

```sql
-- 準備フェーズ
CREATE TABLE transaction_status (
    id INTEGER PRIMARY KEY,
    status TEXT CHECK(status IN ('PREPARING', 'PREPARED', 'COMMITTING', 'COMMITTED', 'ABORTING', 'ABORTED')),
    timestamp TEXT DEFAULT CURRENT_TIMESTAMP
);

-- フェーズ1: 準備
BEGIN;
INSERT INTO transaction_status (status) VALUES ('PREPARING');
-- 各種チェック...
UPDATE transaction_status SET status = 'PREPARED' WHERE id = last_insert_rowid();
COMMIT;

-- フェーズ2: コミット
BEGIN;
UPDATE transaction_status SET status = 'COMMITTING' WHERE id = ?;
-- 実際の更新処理
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE transaction_status SET status = 'COMMITTED' WHERE id = ?;
COMMIT;
```

## 7.10 パフォーマンスとトランザクション

### トランザクションサイズの最適化

```sql
-- パフォーマンステスト用のテーブル
CREATE TABLE performance_test (
    id INTEGER PRIMARY KEY,
    data TEXT
);

-- 方法1: 個別トランザクション（遅い）
-- 約10秒
.timer on
-- 10000件の個別INSERT

-- 方法2: 大きな単一トランザクション（速い）
-- 約0.1秒
BEGIN;
-- 10000件のINSERT
COMMIT;

-- 方法3: 適切なサイズのバッチ（バランスが良い）
-- 1000件ずつのトランザクション
BEGIN;
-- 1000件のINSERT
COMMIT;
-- 繰り返し...
.timer off
```

### 同期モードとパフォーマンス

```sql
-- 同期モードの比較
PRAGMA synchronous = OFF;    -- 最速だが危険
PRAGMA synchronous = NORMAL;  -- バランス型
PRAGMA synchronous = FULL;    -- 最も安全（デフォルト）

-- ベンチマーク
CREATE TABLE sync_test (id INTEGER PRIMARY KEY, data TEXT);

-- 各モードでテスト
.timer on
BEGIN;
-- 1000件のINSERT/UPDATE/DELETE
COMMIT;
.timer off
```

## 7.11 トラブルシューティング

### よくある問題と解決策

```sql
-- 1. database is locked エラー
PRAGMA busy_timeout = 10000;  -- 10秒待機

-- 2. トランザクションのネスト
-- SQLiteは真のネストトランザクションをサポートしない
BEGIN;
-- BEGIN; -- エラー：既にトランザクション内
SAVEPOINT nested;  -- 代わりにSAVEPOINTを使用
-- 処理
RELEASE nested;
COMMIT;

-- 3. 大きなトランザクションによるメモリ使用
PRAGMA cache_size = -64000;  -- 64MBに増加
PRAGMA temp_store = MEMORY;  -- 一時ストレージをメモリに

-- 4. WALファイルの肥大化
PRAGMA wal_checkpoint(TRUNCATE);
PRAGMA journal_size_limit = 67108864;  -- 64MBに制限
```

### デバッグテクニック

```sql
-- トランザクションの状態確認
SELECT * FROM sqlite_master WHERE type = 'table' AND name = 'sqlite_sequence';

-- ロックの状態確認（プログラムから）
-- PRAGMA lock_status;  -- 実装依存

-- トランザクションログの実装
CREATE TABLE transaction_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    transaction_id TEXT,
    operation TEXT,
    table_name TEXT,
    timestamp TEXT DEFAULT CURRENT_TIMESTAMP,
    status TEXT
);

-- トリガーでログ記録
CREATE TRIGGER log_account_changes
AFTER UPDATE ON accounts
BEGIN
    INSERT INTO transaction_log (transaction_id, operation, table_name, status)
    VALUES (
        (SELECT value FROM pragma_compile_options WHERE name = 'TRANSACTION_ID'),
        'UPDATE',
        'accounts',
        'SUCCESS'
    );
END;
```

## 7.12 ベストプラクティス

### トランザクション設計の原則

1. **短く保つ**: トランザクションは必要最小限の時間で完了させる
2. **一貫した順序**: リソースへのアクセス順序を統一してデッドロックを防ぐ
3. **適切な分離**: 読み取り専用操作と更新操作を分離
4. **エラー処理**: 必ずROLLBACKパスを用意
5. **監査証跡**: 重要な操作はログに記録

### 実装例

```sql
-- 理想的なトランザクション処理
CREATE PROCEDURE safe_transfer(
    IN from_id INTEGER,
    IN to_id INTEGER,
    IN amount REAL
)
BEGIN
    DECLARE exit handler for sqlexception
    BEGIN
        ROLLBACK;
        INSERT INTO error_log (message) VALUES ('Transfer failed');
    END;
    
    START TRANSACTION;
    
    -- 検証
    IF amount <= 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Invalid amount';
    END IF;
    
    -- ロック順序を保証（小さいIDから）
    IF from_id < to_id THEN
        UPDATE accounts SET balance = balance - amount WHERE id = from_id;
        UPDATE accounts SET balance = balance + amount WHERE id = to_id;
    ELSE
        UPDATE accounts SET balance = balance + amount WHERE id = to_id;
        UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    END IF;
    
    -- 監査ログ
    INSERT INTO transactions (from_account_id, to_account_id, amount)
    VALUES (from_id, to_id, amount);
    
    COMMIT;
END;
```

## 7.13 まとめ

この章では、SQLiteにおけるトランザクションの概念と実践的な使用方法を学びました。トランザクションは、データの整合性を保ちながら複数の操作を安全に実行するための重要な機能です。

### この章で学んだこと

- トランザクションの基本操作（BEGIN、COMMIT、ROLLBACK）
- ACID特性の理解と実装
- 各種ジャーナルモードとその特徴
- 同時実行制御とロックの仕組み
- パフォーマンスを考慮したトランザクション設計
- エラーハンドリングとトラブルシューティング

### 確認問題

1. ACID特性のそれぞれについて、具体例を挙げて説明してください
2. WALモードとロールバックジャーナルモードの違いと、それぞれの利点を説明してください
3. デッドロックが発生する条件と、それを防ぐ方法を説明してください
4. 大量のデータを挿入する際の、トランザクションを使った最適化方法を説明してください

---

**[← 第6章: インデックスとパフォーマンスへ戻る](./chapter06-indexes-performance.md)** | **[→ 第8章: プログラミング言語との連携へ進む](./chapter08-programming-integration.md)**