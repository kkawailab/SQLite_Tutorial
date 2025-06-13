# 第9章: 実践演習

## 9.1 はじめに

この最終章では、これまでに学んだSQLiteの知識を総合的に活用して、実践的なアプリケーションを構築します。ToDoアプリケーションと在庫管理システムを題材に、設計から実装、最適化まで一連の流れを体験します。

## 9.2 プロジェクト1: ToDoアプリケーション

### 要件定義

以下の機能を持つToDoアプリケーションを作成します：

1. タスクの作成・編集・削除
2. カテゴリによる分類
3. 優先度と期限の管理
4. タスクの検索とフィルタリング
5. 統計情報の表示

### データベース設計

```sql
-- データベースの初期設定
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;

-- カテゴリテーブル
CREATE TABLE categories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL,
    color TEXT DEFAULT '#808080',
    icon TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- タスクテーブル
CREATE TABLE tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    description TEXT,
    category_id INTEGER,
    priority INTEGER DEFAULT 0 CHECK(priority >= 0 AND priority <= 5),
    status TEXT DEFAULT 'pending' CHECK(status IN ('pending', 'in_progress', 'completed', 'cancelled')),
    due_date TEXT,
    completed_at TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL
);

-- タグテーブル
CREATE TABLE tags (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL
);

-- タスクとタグの関連テーブル
CREATE TABLE task_tags (
    task_id INTEGER NOT NULL,
    tag_id INTEGER NOT NULL,
    PRIMARY KEY (task_id, tag_id),
    FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE
);

-- 添付ファイルテーブル
CREATE TABLE attachments (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id INTEGER NOT NULL,
    filename TEXT NOT NULL,
    file_path TEXT NOT NULL,
    file_size INTEGER,
    mime_type TEXT,
    uploaded_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE
);

-- インデックスの作成
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_due_date ON tasks(due_date);
CREATE INDEX idx_tasks_category ON tasks(category_id);
CREATE INDEX idx_tasks_priority ON tasks(priority);
CREATE INDEX idx_task_tags_tag ON task_tags(tag_id);

-- トリガー：更新時刻の自動更新
CREATE TRIGGER update_task_timestamp 
AFTER UPDATE ON tasks
BEGIN
    UPDATE tasks SET updated_at = CURRENT_TIMESTAMP WHERE id = NEW.id;
END;

-- トリガー：タスク完了時の処理
CREATE TRIGGER task_completed 
AFTER UPDATE OF status ON tasks
WHEN NEW.status = 'completed' AND OLD.status != 'completed'
BEGIN
    UPDATE tasks SET completed_at = CURRENT_TIMESTAMP WHERE id = NEW.id;
END;

-- ビュー：タスクの詳細情報
CREATE VIEW v_task_details AS
SELECT 
    t.id,
    t.title,
    t.description,
    t.priority,
    t.status,
    t.due_date,
    t.created_at,
    t.updated_at,
    t.completed_at,
    c.name AS category_name,
    c.color AS category_color,
    GROUP_CONCAT(tg.name) AS tags,
    COUNT(DISTINCT a.id) AS attachment_count,
    CASE 
        WHEN t.due_date < date('now') AND t.status != 'completed' THEN 'overdue'
        WHEN t.due_date = date('now') AND t.status != 'completed' THEN 'due_today'
        ELSE 'normal'
    END AS urgency
FROM tasks t
LEFT JOIN categories c ON t.category_id = c.id
LEFT JOIN task_tags tt ON t.id = tt.task_id
LEFT JOIN tags tg ON tt.tag_id = tg.id
LEFT JOIN attachments a ON t.id = a.task_id
GROUP BY t.id;
```

### Pythonでの実装

```python
import sqlite3
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
from contextlib import contextmanager
import json

class TodoApp:
    def __init__(self, db_path='todo.db'):
        self.db_path = db_path
        self._initialize_database()
    
    @contextmanager
    def get_db(self):
        """データベース接続のコンテキストマネージャー"""
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        conn.execute("PRAGMA foreign_keys = ON")
        try:
            yield conn
            conn.commit()
        except Exception:
            conn.rollback()
            raise
        finally:
            conn.close()
    
    def _initialize_database(self):
        """データベースの初期化（上記のSQLを実行）"""
        with self.get_db() as conn:
            # ここに上記のCREATE文を実行
            pass
    
    # タスク操作
    def create_task(self, title: str, description: str = None, 
                   category_id: int = None, priority: int = 0, 
                   due_date: str = None, tags: List[str] = None) -> int:
        """新しいタスクを作成"""
        with self.get_db() as conn:
            cursor = conn.execute(
                """INSERT INTO tasks (title, description, category_id, priority, due_date)
                   VALUES (?, ?, ?, ?, ?)""",
                (title, description, category_id, priority, due_date)
            )
            task_id = cursor.lastrowid
            
            # タグの処理
            if tags:
                for tag_name in tags:
                    # タグが存在しない場合は作成
                    cursor = conn.execute(
                        "INSERT OR IGNORE INTO tags (name) VALUES (?)",
                        (tag_name,)
                    )
                    # タグIDを取得
                    tag_id = conn.execute(
                        "SELECT id FROM tags WHERE name = ?",
                        (tag_name,)
                    ).fetchone()[0]
                    # タスクとタグを関連付け
                    conn.execute(
                        "INSERT INTO task_tags (task_id, tag_id) VALUES (?, ?)",
                        (task_id, tag_id)
                    )
            
            return task_id
    
    def update_task_status(self, task_id: int, status: str):
        """タスクのステータスを更新"""
        valid_statuses = ['pending', 'in_progress', 'completed', 'cancelled']
        if status not in valid_statuses:
            raise ValueError(f"Invalid status. Must be one of {valid_statuses}")
        
        with self.get_db() as conn:
            conn.execute(
                "UPDATE tasks SET status = ? WHERE id = ?",
                (status, task_id)
            )
    
    def get_tasks_by_status(self, status: str) -> List[Dict]:
        """ステータスでタスクを取得"""
        with self.get_db() as conn:
            rows = conn.execute(
                """SELECT * FROM v_task_details 
                   WHERE status = ? 
                   ORDER BY priority DESC, due_date ASC""",
                (status,)
            ).fetchall()
            return [dict(row) for row in rows]
    
    def get_overdue_tasks(self) -> List[Dict]:
        """期限切れのタスクを取得"""
        with self.get_db() as conn:
            rows = conn.execute(
                """SELECT * FROM v_task_details 
                   WHERE urgency = 'overdue' 
                   ORDER BY due_date ASC"""
            ).fetchall()
            return [dict(row) for row in rows]
    
    def search_tasks(self, query: str) -> List[Dict]:
        """タスクを検索"""
        with self.get_db() as conn:
            search_pattern = f"%{query}%"
            rows = conn.execute(
                """SELECT * FROM v_task_details 
                   WHERE title LIKE ? OR description LIKE ? OR tags LIKE ?
                   ORDER BY created_at DESC""",
                (search_pattern, search_pattern, search_pattern)
            ).fetchall()
            return [dict(row) for row in rows]
    
    # カテゴリ操作
    def create_category(self, name: str, color: str = '#808080', icon: str = None) -> int:
        """カテゴリを作成"""
        with self.get_db() as conn:
            cursor = conn.execute(
                "INSERT INTO categories (name, color, icon) VALUES (?, ?, ?)",
                (name, color, icon)
            )
            return cursor.lastrowid
    
    def get_categories_with_task_count(self) -> List[Dict]:
        """カテゴリとタスク数を取得"""
        with self.get_db() as conn:
            rows = conn.execute(
                """SELECT 
                       c.id,
                       c.name,
                       c.color,
                       c.icon,
                       COUNT(t.id) as task_count,
                       COUNT(CASE WHEN t.status = 'completed' THEN 1 END) as completed_count
                   FROM categories c
                   LEFT JOIN tasks t ON c.id = t.category_id
                   GROUP BY c.id
                   ORDER BY c.name"""
            ).fetchall()
            return [dict(row) for row in rows]
    
    # 統計情報
    def get_statistics(self) -> Dict:
        """統計情報を取得"""
        with self.get_db() as conn:
            # 基本統計
            stats = conn.execute(
                """SELECT 
                       COUNT(*) as total_tasks,
                       COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed_tasks,
                       COUNT(CASE WHEN status = 'pending' THEN 1 END) as pending_tasks,
                       COUNT(CASE WHEN status = 'in_progress' THEN 1 END) as in_progress_tasks,
                       COUNT(CASE WHEN due_date < date('now') AND status != 'completed' THEN 1 END) as overdue_tasks
                   FROM tasks"""
            ).fetchone()
            
            # 今週の完了タスク数
            week_completed = conn.execute(
                """SELECT COUNT(*) as count
                   FROM tasks
                   WHERE status = 'completed' 
                     AND completed_at >= date('now', '-7 days')"""
            ).fetchone()['count']
            
            # カテゴリ別統計
            category_stats = conn.execute(
                """SELECT 
                       c.name,
                       COUNT(t.id) as task_count,
                       AVG(CASE WHEN t.status = 'completed' THEN 1.0 ELSE 0.0 END) * 100 as completion_rate
                   FROM categories c
                   LEFT JOIN tasks t ON c.id = t.category_id
                   GROUP BY c.id
                   HAVING task_count > 0"""
            ).fetchall()
            
            return {
                'total_tasks': stats['total_tasks'],
                'completed_tasks': stats['completed_tasks'],
                'pending_tasks': stats['pending_tasks'],
                'in_progress_tasks': stats['in_progress_tasks'],
                'overdue_tasks': stats['overdue_tasks'],
                'completion_rate': (stats['completed_tasks'] / stats['total_tasks'] * 100) if stats['total_tasks'] > 0 else 0,
                'week_completed': week_completed,
                'category_stats': [dict(row) for row in category_stats]
            }
    
    # 高度な機能
    def bulk_update_tasks(self, task_updates: List[Tuple[int, Dict]]):
        """複数タスクの一括更新"""
        with self.get_db() as conn:
            for task_id, updates in task_updates:
                set_clause = ', '.join(f"{k} = ?" for k in updates.keys())
                values = list(updates.values()) + [task_id]
                conn.execute(
                    f"UPDATE tasks SET {set_clause} WHERE id = ?",
                    values
                )
    
    def export_tasks(self, format='json') -> str:
        """タスクをエクスポート"""
        with self.get_db() as conn:
            tasks = conn.execute("SELECT * FROM v_task_details").fetchall()
            
            if format == 'json':
                return json.dumps([dict(task) for task in tasks], indent=2)
            elif format == 'csv':
                import csv
                import io
                output = io.StringIO()
                writer = csv.DictWriter(output, fieldnames=tasks[0].keys())
                writer.writeheader()
                for task in tasks:
                    writer.writerow(dict(task))
                return output.getvalue()
            else:
                raise ValueError(f"Unsupported format: {format}")

# 使用例
def demo_todo_app():
    app = TodoApp()
    
    # カテゴリ作成
    work_id = app.create_category("仕事", "#FF5733", "💼")
    personal_id = app.create_category("個人", "#33FF57", "🏠")
    
    # タスク作成
    task1 = app.create_task(
        "プロジェクト提案書の作成",
        "Q4の新規プロジェクトの提案書を作成する",
        category_id=work_id,
        priority=5,
        due_date="2024-12-20",
        tags=["重要", "プロジェクト"]
    )
    
    task2 = app.create_task(
        "買い物",
        "週末の買い物リスト",
        category_id=personal_id,
        priority=2,
        due_date="2024-12-15",
        tags=["日常"]
    )
    
    # ステータス更新
    app.update_task_status(task1, 'in_progress')
    
    # タスク検索
    results = app.search_tasks("プロジェクト")
    print(f"検索結果: {len(results)}件")
    
    # 統計情報
    stats = app.get_statistics()
    print(f"統計情報: {stats}")

# demo_todo_app()
```

## 9.3 プロジェクト2: 在庫管理システム

### 要件定義

以下の機能を持つ在庫管理システムを構築します：

1. 商品マスタ管理
2. 入出荷管理
3. 在庫数のリアルタイム追跡
4. 在庫アラート機能
5. 棚卸し機能
6. レポート生成

### データベース設計

```sql
-- 商品カテゴリ
CREATE TABLE product_categories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL,
    parent_id INTEGER,
    FOREIGN KEY (parent_id) REFERENCES product_categories(id)
);

-- 仕入先
CREATE TABLE suppliers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    contact_person TEXT,
    email TEXT,
    phone TEXT,
    address TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- 商品マスタ
CREATE TABLE products (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sku TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    category_id INTEGER,
    supplier_id INTEGER,
    unit_price REAL NOT NULL CHECK(unit_price >= 0),
    cost_price REAL CHECK(cost_price >= 0),
    min_stock INTEGER DEFAULT 0,
    max_stock INTEGER DEFAULT 1000,
    reorder_point INTEGER DEFAULT 10,
    is_active BOOLEAN DEFAULT 1,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES product_categories(id),
    FOREIGN KEY (supplier_id) REFERENCES suppliers(id)
);

-- 倉庫/保管場所
CREATE TABLE warehouses (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    location TEXT,
    capacity INTEGER
);

-- 在庫
CREATE TABLE inventory (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id INTEGER NOT NULL,
    warehouse_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 0 CHECK(quantity >= 0),
    reserved_quantity INTEGER DEFAULT 0 CHECK(reserved_quantity >= 0),
    last_updated TEXT DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (product_id, warehouse_id),
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id)
);

-- 在庫移動履歴
CREATE TABLE inventory_transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id INTEGER NOT NULL,
    warehouse_id INTEGER NOT NULL,
    transaction_type TEXT NOT NULL CHECK(transaction_type IN ('in', 'out', 'transfer', 'adjustment')),
    quantity INTEGER NOT NULL,
    reference_type TEXT,  -- 'purchase_order', 'sales_order', 'transfer', 'adjustment'
    reference_id INTEGER,
    notes TEXT,
    user_id INTEGER,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id)
);

-- 発注
CREATE TABLE purchase_orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    supplier_id INTEGER NOT NULL,
    order_date TEXT DEFAULT CURRENT_TIMESTAMP,
    expected_date TEXT,
    status TEXT DEFAULT 'pending' CHECK(status IN ('pending', 'ordered', 'received', 'cancelled')),
    total_amount REAL,
    notes TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(id)
);

-- 発注明細
CREATE TABLE purchase_order_items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    purchase_order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL CHECK(quantity > 0),
    unit_price REAL NOT NULL CHECK(unit_price >= 0),
    received_quantity INTEGER DEFAULT 0,
    FOREIGN KEY (purchase_order_id) REFERENCES purchase_orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);

-- インデックス
CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_warehouse ON inventory(warehouse_id);
CREATE INDEX idx_transactions_product ON inventory_transactions(product_id);
CREATE INDEX idx_transactions_date ON inventory_transactions(created_at);

-- トリガー：在庫数の自動更新
CREATE TRIGGER update_inventory_on_transaction
AFTER INSERT ON inventory_transactions
BEGIN
    -- 入庫
    UPDATE inventory 
    SET quantity = quantity + NEW.quantity,
        last_updated = CURRENT_TIMESTAMP
    WHERE product_id = NEW.product_id 
      AND warehouse_id = NEW.warehouse_id
      AND NEW.transaction_type = 'in';
    
    -- 出庫
    UPDATE inventory 
    SET quantity = quantity - NEW.quantity,
        last_updated = CURRENT_TIMESTAMP
    WHERE product_id = NEW.product_id 
      AND warehouse_id = NEW.warehouse_id
      AND NEW.transaction_type = 'out';
END;

-- ビュー：在庫状況サマリー
CREATE VIEW v_inventory_summary AS
SELECT 
    p.id,
    p.sku,
    p.name,
    pc.name as category,
    SUM(i.quantity) as total_quantity,
    SUM(i.reserved_quantity) as total_reserved,
    SUM(i.quantity - i.reserved_quantity) as available_quantity,
    p.min_stock,
    p.reorder_point,
    CASE 
        WHEN SUM(i.quantity) <= p.min_stock THEN 'critical'
        WHEN SUM(i.quantity) <= p.reorder_point THEN 'low'
        ELSE 'normal'
    END as stock_status,
    p.unit_price * SUM(i.quantity) as inventory_value
FROM products p
LEFT JOIN inventory i ON p.id = i.product_id
LEFT JOIN product_categories pc ON p.category_id = pc.id
WHERE p.is_active = 1
GROUP BY p.id;

-- ビュー：在庫アラート
CREATE VIEW v_inventory_alerts AS
SELECT 
    p.id,
    p.sku,
    p.name,
    SUM(i.quantity) as current_stock,
    p.reorder_point,
    p.min_stock,
    s.name as supplier_name,
    s.email as supplier_email
FROM products p
LEFT JOIN inventory i ON p.id = i.product_id
LEFT JOIN suppliers s ON p.supplier_id = s.id
WHERE p.is_active = 1
GROUP BY p.id
HAVING current_stock <= p.reorder_point;
```

### 在庫管理システムの実装

```python
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
import sqlite3
from contextlib import contextmanager

class InventoryManagementSystem:
    def __init__(self, db_path='inventory.db'):
        self.db_path = db_path
        self._initialize_database()
    
    @contextmanager
    def get_db(self):
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        conn.execute("PRAGMA foreign_keys = ON")
        try:
            yield conn
            conn.commit()
        except Exception:
            conn.rollback()
            raise
        finally:
            conn.close()
    
    def _initialize_database(self):
        """データベースの初期化"""
        # 上記のSQL文を実行
        pass
    
    # 商品管理
    def create_product(self, sku: str, name: str, unit_price: float,
                      category_id: int = None, supplier_id: int = None,
                      min_stock: int = 0, reorder_point: int = 10) -> int:
        """商品を登録"""
        with self.get_db() as conn:
            cursor = conn.execute(
                """INSERT INTO products 
                   (sku, name, unit_price, category_id, supplier_id, min_stock, reorder_point)
                   VALUES (?, ?, ?, ?, ?, ?, ?)""",
                (sku, name, unit_price, category_id, supplier_id, min_stock, reorder_point)
            )
            
            product_id = cursor.lastrowid
            
            # 各倉庫に初期在庫レコードを作成
            warehouses = conn.execute("SELECT id FROM warehouses").fetchall()
            for warehouse in warehouses:
                conn.execute(
                    """INSERT OR IGNORE INTO inventory (product_id, warehouse_id, quantity)
                       VALUES (?, ?, 0)""",
                    (product_id, warehouse['id'])
                )
            
            return product_id
    
    # 在庫操作
    def receive_stock(self, product_id: int, warehouse_id: int, quantity: int,
                     purchase_order_id: int = None, notes: str = None):
        """入庫処理"""
        with self.get_db() as conn:
            # 在庫トランザクション記録
            conn.execute(
                """INSERT INTO inventory_transactions 
                   (product_id, warehouse_id, transaction_type, quantity, 
                    reference_type, reference_id, notes)
                   VALUES (?, ?, 'in', ?, 'purchase_order', ?, ?)""",
                (product_id, warehouse_id, quantity, purchase_order_id, notes)
            )
            
            # 在庫レコードが存在しない場合は作成
            conn.execute(
                """INSERT OR IGNORE INTO inventory (product_id, warehouse_id, quantity)
                   VALUES (?, ?, 0)""",
                (product_id, warehouse_id)
            )
    
    def ship_stock(self, product_id: int, warehouse_id: int, quantity: int,
                  order_id: int = None, notes: str = None) -> bool:
        """出庫処理"""
        with self.get_db() as conn:
            # 在庫確認
            current = conn.execute(
                """SELECT quantity - reserved_quantity as available 
                   FROM inventory 
                   WHERE product_id = ? AND warehouse_id = ?""",
                (product_id, warehouse_id)
            ).fetchone()
            
            if not current or current['available'] < quantity:
                raise ValueError("在庫が不足しています")
            
            # 出庫トランザクション記録
            conn.execute(
                """INSERT INTO inventory_transactions 
                   (product_id, warehouse_id, transaction_type, quantity,
                    reference_type, reference_id, notes)
                   VALUES (?, ?, 'out', ?, 'sales_order', ?, ?)""",
                (product_id, warehouse_id, quantity, order_id, notes)
            )
            
            return True
    
    def transfer_stock(self, product_id: int, from_warehouse: int, 
                      to_warehouse: int, quantity: int) -> bool:
        """倉庫間移動"""
        with self.get_db() as conn:
            # 出庫
            self.ship_stock(product_id, from_warehouse, quantity, 
                          notes=f"Transfer to warehouse {to_warehouse}")
            
            # 入庫
            self.receive_stock(product_id, to_warehouse, quantity,
                             notes=f"Transfer from warehouse {from_warehouse}")
            
            return True
    
    # 発注管理
    def create_purchase_order(self, supplier_id: int, items: List[Dict]) -> int:
        """発注書作成"""
        with self.get_db() as conn:
            # 発注書作成
            cursor = conn.execute(
                """INSERT INTO purchase_orders (supplier_id, status)
                   VALUES (?, 'pending')""",
                (supplier_id,)
            )
            order_id = cursor.lastrowid
            
            # 明細追加
            total_amount = 0
            for item in items:
                conn.execute(
                    """INSERT INTO purchase_order_items 
                       (purchase_order_id, product_id, quantity, unit_price)
                       VALUES (?, ?, ?, ?)""",
                    (order_id, item['product_id'], item['quantity'], item['unit_price'])
                )
                total_amount += item['quantity'] * item['unit_price']
            
            # 合計金額更新
            conn.execute(
                "UPDATE purchase_orders SET total_amount = ? WHERE id = ?",
                (total_amount, order_id)
            )
            
            return order_id
    
    def get_reorder_suggestions(self) -> List[Dict]:
        """発注提案を取得"""
        with self.get_db() as conn:
            suggestions = conn.execute("""
                SELECT 
                    v.id,
                    v.sku,
                    v.name,
                    v.current_stock,
                    v.reorder_point,
                    v.supplier_name,
                    p.cost_price,
                    CASE 
                        WHEN v.current_stock = 0 THEN v.reorder_point * 2
                        ELSE v.reorder_point - v.current_stock + (v.reorder_point / 2)
                    END as suggested_quantity
                FROM v_inventory_alerts v
                JOIN products p ON v.id = p.id
                ORDER BY v.current_stock ASC
            """).fetchall()
            
            return [dict(s) for s in suggestions]
    
    # レポート機能
    def get_inventory_valuation(self) -> Dict:
        """在庫評価額を計算"""
        with self.get_db() as conn:
            # カテゴリ別評価額
            category_valuation = conn.execute("""
                SELECT 
                    pc.name as category,
                    COUNT(DISTINCT p.id) as product_count,
                    SUM(i.quantity) as total_units,
                    SUM(i.quantity * p.unit_price) as retail_value,
                    SUM(i.quantity * p.cost_price) as cost_value
                FROM inventory i
                JOIN products p ON i.product_id = p.id
                JOIN product_categories pc ON p.category_id = pc.id
                GROUP BY pc.id
            """).fetchall()
            
            # 倉庫別評価額
            warehouse_valuation = conn.execute("""
                SELECT 
                    w.name as warehouse,
                    COUNT(DISTINCT i.product_id) as product_types,
                    SUM(i.quantity) as total_units,
                    SUM(i.quantity * p.unit_price) as value
                FROM inventory i
                JOIN products p ON i.product_id = p.id
                JOIN warehouses w ON i.warehouse_id = w.id
                WHERE i.quantity > 0
                GROUP BY w.id
            """).fetchall()
            
            # 総評価額
            total = conn.execute("""
                SELECT 
                    SUM(i.quantity * p.unit_price) as total_retail_value,
                    SUM(i.quantity * p.cost_price) as total_cost_value
                FROM inventory i
                JOIN products p ON i.product_id = p.id
            """).fetchone()
            
            return {
                'total_retail_value': total['total_retail_value'] or 0,
                'total_cost_value': total['total_cost_value'] or 0,
                'by_category': [dict(c) for c in category_valuation],
                'by_warehouse': [dict(w) for w in warehouse_valuation]
            }
    
    def get_movement_report(self, start_date: str, end_date: str) -> List[Dict]:
        """期間内の在庫移動レポート"""
        with self.get_db() as conn:
            movements = conn.execute("""
                SELECT 
                    p.sku,
                    p.name,
                    it.transaction_type,
                    SUM(CASE WHEN it.transaction_type = 'in' THEN it.quantity ELSE 0 END) as total_in,
                    SUM(CASE WHEN it.transaction_type = 'out' THEN it.quantity ELSE 0 END) as total_out,
                    COUNT(*) as transaction_count
                FROM inventory_transactions it
                JOIN products p ON it.product_id = p.id
                WHERE it.created_at BETWEEN ? AND ?
                GROUP BY p.id, it.transaction_type
                ORDER BY p.name
            """, (start_date, end_date)).fetchall()
            
            return [dict(m) for m in movements]
    
    def perform_stock_count(self, counts: List[Dict]):
        """棚卸し処理"""
        with self.get_db() as conn:
            adjustments = []
            
            for count in counts:
                # 現在の在庫数を取得
                current = conn.execute(
                    """SELECT quantity FROM inventory 
                       WHERE product_id = ? AND warehouse_id = ?""",
                    (count['product_id'], count['warehouse_id'])
                ).fetchone()
                
                if current:
                    difference = count['counted_quantity'] - current['quantity']
                    
                    if difference != 0:
                        # 調整トランザクションを記録
                        transaction_type = 'in' if difference > 0 else 'out'
                        conn.execute(
                            """INSERT INTO inventory_transactions 
                               (product_id, warehouse_id, transaction_type, quantity,
                                reference_type, notes)
                               VALUES (?, ?, ?, ?, 'adjustment', ?)""",
                            (count['product_id'], count['warehouse_id'], 
                             transaction_type, abs(difference),
                             f"Stock count adjustment: {difference:+d}")
                        )
                        
                        adjustments.append({
                            'product_id': count['product_id'],
                            'warehouse_id': count['warehouse_id'],
                            'previous': current['quantity'],
                            'counted': count['counted_quantity'],
                            'adjustment': difference
                        })
            
            return adjustments

# 使用例とテスト
def demo_inventory_system():
    ims = InventoryManagementSystem()
    
    # 初期データ設定
    # 倉庫作成
    with ims.get_db() as conn:
        conn.execute("INSERT INTO warehouses (name, location, capacity) VALUES (?, ?, ?)",
                    ("メイン倉庫", "東京", 10000))
        conn.execute("INSERT INTO warehouses (name, location, capacity) VALUES (?, ?, ?)",
                    ("サブ倉庫", "大阪", 5000))
        
        # カテゴリ作成
        conn.execute("INSERT INTO product_categories (name) VALUES (?)", ("電子機器",))
        conn.execute("INSERT INTO product_categories (name) VALUES (?)", ("書籍",))
        
        # 仕入先作成
        conn.execute("""INSERT INTO suppliers (name, email, phone) 
                       VALUES (?, ?, ?)""",
                    ("テックサプライヤー", "tech@supplier.com", "03-1234-5678"))
    
    # 商品登録
    product_id = ims.create_product(
        sku="LAPTOP-001",
        name="ノートパソコン Pro",
        unit_price=150000,
        category_id=1,
        supplier_id=1,
        min_stock=5,
        reorder_point=10
    )
    
    # 入庫
    ims.receive_stock(product_id, 1, 20, notes="初期在庫")
    
    # 出庫
    try:
        ims.ship_stock(product_id, 1, 5, notes="顧客注文 #1234")
    except ValueError as e:
        print(f"出庫エラー: {e}")
    
    # 在庫評価レポート
    valuation = ims.get_inventory_valuation()
    print(f"在庫総評価額: ¥{valuation['total_retail_value']:,.0f}")
    
    # 発注提案
    suggestions = ims.get_reorder_suggestions()
    print(f"発注提案: {len(suggestions)}件")

# demo_inventory_system()
```

## 9.4 ベストプラクティス

### データベース設計のポイント

1. **正規化と非正規化のバランス**
   - 基本的には第3正規形まで正規化
   - パフォーマンスが必要な場合は適度に非正規化

2. **インデックス戦略**
   - 外部キーには必ずインデックス
   - 検索条件によく使われる列にインデックス
   - 複合インデックスは使用頻度の高い順

3. **制約の活用**
   - CHECK制約でデータの妥当性を保証
   - FOREIGN KEYで参照整合性を維持
   - UNIQUEで重複を防止

### エラーハンドリング

```python
class DatabaseError(Exception):
    """データベース関連の基底例外"""
    pass

class NotFoundError(DatabaseError):
    """レコードが見つからない"""
    pass

class ValidationError(DatabaseError):
    """バリデーションエラー"""
    pass

class InsufficientStockError(DatabaseError):
    """在庫不足エラー"""
    pass

def with_error_handling(func):
    """エラーハンドリングデコレータ"""
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except sqlite3.IntegrityError as e:
            if "UNIQUE constraint failed" in str(e):
                raise ValidationError("既に登録されています")
            elif "FOREIGN KEY constraint failed" in str(e):
                raise ValidationError("参照先が存在しません")
            raise DatabaseError(str(e))
        except sqlite3.OperationalError as e:
            raise DatabaseError(f"データベース操作エラー: {e}")
    return wrapper
```

### パフォーマンス最適化

```python
# バルクインサート最適化
def bulk_insert_optimized(conn, table, records):
    """最適化されたバルクインサート"""
    # トランザクション内で実行
    conn.execute("BEGIN")
    
    # 一時的にチェックを無効化
    conn.execute("PRAGMA synchronous = OFF")
    conn.execute("PRAGMA journal_mode = MEMORY")
    
    try:
        # プレースホルダーを動的に生成
        columns = records[0].keys()
        placeholders = ','.join(['?' for _ in columns])
        column_names = ','.join(columns)
        
        sql = f"INSERT INTO {table} ({column_names}) VALUES ({placeholders})"
        
        # executemanyで一括実行
        conn.executemany(sql, [tuple(r.values()) for r in records])
        
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        # 設定を戻す
        conn.execute("PRAGMA synchronous = FULL")
        conn.execute("PRAGMA journal_mode = DELETE")
```

### テストとデバッグ

```python
import unittest
from datetime import datetime

class TestInventorySystem(unittest.TestCase):
    def setUp(self):
        """各テストの前に実行"""
        self.ims = InventoryManagementSystem(':memory:')
        self._setup_test_data()
    
    def _setup_test_data(self):
        """テストデータの準備"""
        with self.ims.get_db() as conn:
            conn.execute("INSERT INTO warehouses (id, name) VALUES (1, 'Test Warehouse')")
            conn.execute("INSERT INTO suppliers (id, name) VALUES (1, 'Test Supplier')")
            conn.execute("INSERT INTO product_categories (id, name) VALUES (1, 'Test Category')")
    
    def test_create_product(self):
        """商品作成のテスト"""
        product_id = self.ims.create_product(
            sku="TEST-001",
            name="テスト商品",
            unit_price=1000,
            category_id=1,
            supplier_id=1
        )
        self.assertIsNotNone(product_id)
        
        # 商品が作成されたか確認
        with self.ims.get_db() as conn:
            product = conn.execute(
                "SELECT * FROM products WHERE id = ?", 
                (product_id,)
            ).fetchone()
            self.assertEqual(product['sku'], "TEST-001")
    
    def test_stock_movement(self):
        """在庫移動のテスト"""
        # 商品作成
        product_id = self.ims.create_product("TEST-002", "テスト商品2", 2000)
        
        # 入庫
        self.ims.receive_stock(product_id, 1, 100)
        
        # 在庫確認
        with self.ims.get_db() as conn:
            stock = conn.execute(
                "SELECT quantity FROM inventory WHERE product_id = ?",
                (product_id,)
            ).fetchone()
            self.assertEqual(stock['quantity'], 100)
        
        # 出庫
        self.ims.ship_stock(product_id, 1, 30)
        
        # 在庫確認
        with self.ims.get_db() as conn:
            stock = conn.execute(
                "SELECT quantity FROM inventory WHERE product_id = ?",
                (product_id,)
            ).fetchone()
            self.assertEqual(stock['quantity'], 70)
    
    def test_insufficient_stock(self):
        """在庫不足のテスト"""
        product_id = self.ims.create_product("TEST-003", "テスト商品3", 3000)
        self.ims.receive_stock(product_id, 1, 10)
        
        # 在庫以上の出庫を試みる
        with self.assertRaises(ValueError):
            self.ims.ship_stock(product_id, 1, 20)

if __name__ == '__main__':
    unittest.main()
```

## 9.5 トラブルシューティング

### よくある問題と解決方法

#### 1. データベースロック

```python
# タイムアウト設定
conn.execute("PRAGMA busy_timeout = 5000")  # 5秒

# WALモードの使用
conn.execute("PRAGMA journal_mode = WAL")
```

#### 2. パフォーマンス問題

```python
# クエリの実行計画を確認
def analyze_query(conn, query, params=()):
    plan = conn.execute(f"EXPLAIN QUERY PLAN {query}", params).fetchall()
    for row in plan:
        print(row)

# インデックスの効果を測定
def measure_query_time(conn, query, params=()):
    import time
    start = time.time()
    conn.execute(query, params).fetchall()
    return time.time() - start
```

#### 3. データ整合性

```python
# データ整合性チェック
def check_data_integrity(conn):
    """データ整合性をチェック"""
    issues = []
    
    # 孤立したレコードをチェック
    orphaned = conn.execute("""
        SELECT COUNT(*) as count
        FROM inventory i
        LEFT JOIN products p ON i.product_id = p.id
        WHERE p.id IS NULL
    """).fetchone()
    
    if orphaned['count'] > 0:
        issues.append(f"孤立した在庫レコード: {orphaned['count']}件")
    
    # 在庫数の不整合をチェック
    inconsistent = conn.execute("""
        SELECT p.id, p.name, i.quantity as inventory_qty,
               COALESCE(SUM(CASE WHEN it.transaction_type = 'in' THEN it.quantity
                                 WHEN it.transaction_type = 'out' THEN -it.quantity
                                 ELSE 0 END), 0) as calculated_qty
        FROM products p
        JOIN inventory i ON p.id = i.product_id
        LEFT JOIN inventory_transactions it ON p.id = it.product_id 
                                            AND i.warehouse_id = it.warehouse_id
        GROUP BY p.id, i.warehouse_id, i.quantity
        HAVING inventory_qty != calculated_qty
    """).fetchall()
    
    if inconsistent:
        issues.append(f"在庫数不整合: {len(inconsistent)}件")
    
    return issues
```

## 9.6 まとめ

この章では、実践的な2つのプロジェクトを通じて、SQLiteを使用したアプリケーション開発の全体像を学びました。

### 学んだこと

1. **要件定義からデータベース設計**
   - 正規化とパフォーマンスのバランス
   - 適切なインデックスの設計
   - 制約による データ整合性の確保

2. **実装のベストプラクティス**
   - エラーハンドリング
   - トランザクション管理
   - パフォーマンス最適化

3. **テストとデバッグ**
   - ユニットテストの作成
   - パフォーマンス測定
   - データ整合性チェック

### 次のステップ

1. **セキュリティの強化**
   - SQLインジェクション対策
   - アクセス制御
   - 監査ログ

2. **スケーラビリティ**
   - シャーディング戦略
   - キャッシュの活用
   - 読み取り専用レプリカ

3. **高度な機能**
   - 全文検索（FTS）
   - 空間データ（R-Tree）
   - カスタム関数

### 最終確認問題

1. ToDoアプリケーションに「繰り返しタスク」機能を追加する場合、どのようなテーブル設計が必要か考えてください
2. 在庫管理システムで「ロット管理」を実装する場合の設計を提案してください
3. 両システムで「マルチテナント」対応する場合の設計変更点を説明してください
4. パフォーマンスボトルネックを特定し改善する手順を説明してください

これでSQLiteチュートリアルは終了です。ここで学んだ知識を活かして、より高度で実用的なアプリケーションの開発に挑戦してください！

---

**[← 第8章: プログラミング言語との連携へ戻る](./chapter08-programming-integration.md)** | **[→ 目次へ戻る](./README.md)**