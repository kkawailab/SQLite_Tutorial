# 第8章: プログラミング言語との連携

## 8.1 はじめに

この章では、様々なプログラミング言語からSQLiteを使用する方法を学びます。Python、JavaScript（Node.js）、Java、C#などの主要な言語での実装例を通じて、実際のアプリケーション開発でSQLiteを活用する方法を習得します。

## 8.2 Python での SQLite 使用

### 基本的な接続と操作

```python
import sqlite3
import json
from datetime import datetime
from contextlib import contextmanager

# データベース接続のコンテキストマネージャー
@contextmanager
def get_db_connection(db_path):
    """データベース接続を管理するコンテキストマネージャー"""
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row  # 結果を辞書風にアクセス可能に
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

# 基本的な使用例
def basic_example():
    # メモリデータベースで例示
    conn = sqlite3.connect(':memory:')
    cursor = conn.cursor()
    
    # テーブル作成
    cursor.execute('''
        CREATE TABLE users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT UNIQUE NOT NULL,
            created_at TEXT DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # データ挿入
    cursor.execute(
        "INSERT INTO users (name, email) VALUES (?, ?)",
        ("山田太郎", "yamada@example.com")
    )
    
    # データ取得
    cursor.execute("SELECT * FROM users")
    for row in cursor.fetchall():
        print(f"ID: {row[0]}, Name: {row[1]}, Email: {row[2]}")
    
    conn.close()

# 実行
basic_example()
```

### クラスベースのデータベース操作

```python
class UserDatabase:
    """ユーザーデータベースを管理するクラス"""
    
    def __init__(self, db_path):
        self.db_path = db_path
        self._create_tables()
    
    def _create_tables(self):
        """必要なテーブルを作成"""
        with get_db_connection(self.db_path) as conn:
            conn.execute('''
                CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT UNIQUE NOT NULL,
                    email TEXT UNIQUE NOT NULL,
                    password_hash TEXT NOT NULL,
                    is_active BOOLEAN DEFAULT 1,
                    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
                    updated_at TEXT DEFAULT CURRENT_TIMESTAMP
                )
            ''')
            
            conn.execute('''
                CREATE INDEX IF NOT EXISTS idx_users_email 
                ON users(email)
            ''')
    
    def create_user(self, username, email, password_hash):
        """新規ユーザーを作成"""
        with get_db_connection(self.db_path) as conn:
            cursor = conn.execute(
                '''INSERT INTO users (username, email, password_hash) 
                   VALUES (?, ?, ?)''',
                (username, email, password_hash)
            )
            return cursor.lastrowid
    
    def get_user_by_email(self, email):
        """メールアドレスでユーザーを検索"""
        with get_db_connection(self.db_path) as conn:
            result = conn.execute(
                "SELECT * FROM users WHERE email = ?", 
                (email,)
            ).fetchone()
            
            if result:
                return dict(result)
            return None
    
    def update_user(self, user_id, **kwargs):
        """ユーザー情報を更新"""
        allowed_fields = ['username', 'email', 'is_active']
        updates = {k: v for k, v in kwargs.items() if k in allowed_fields}
        
        if not updates:
            return False
        
        set_clause = ', '.join(f"{k} = ?" for k in updates.keys())
        values = list(updates.values()) + [user_id]
        
        with get_db_connection(self.db_path) as conn:
            cursor = conn.execute(
                f"UPDATE users SET {set_clause}, updated_at = CURRENT_TIMESTAMP WHERE id = ?",
                values
            )
            return cursor.rowcount > 0
    
    def search_users(self, query, limit=10):
        """ユーザーを検索"""
        with get_db_connection(self.db_path) as conn:
            results = conn.execute(
                '''SELECT id, username, email, created_at 
                   FROM users 
                   WHERE username LIKE ? OR email LIKE ?
                   LIMIT ?''',
                (f'%{query}%', f'%{query}%', limit)
            ).fetchall()
            
            return [dict(row) for row in results]

# 使用例
db = UserDatabase('users.db')
user_id = db.create_user('tanaka', 'tanaka@example.com', 'hashed_password')
print(f"Created user with ID: {user_id}")

user = db.get_user_by_email('tanaka@example.com')
print(f"Found user: {user}")

db.update_user(user_id, username='tanaka_updated')
results = db.search_users('tanaka')
print(f"Search results: {results}")
```

### 高度な機能の実装

```python
import asyncio
import aiosqlite
from typing import List, Dict, Any, Optional
import logging

# 非同期データベース操作
class AsyncUserDatabase:
    """非同期でデータベース操作を行うクラス"""
    
    def __init__(self, db_path: str):
        self.db_path = db_path
        self.logger = logging.getLogger(__name__)
    
    async def initialize(self):
        """データベースの初期化"""
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute('''
                CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT UNIQUE NOT NULL,
                    email TEXT UNIQUE NOT NULL,
                    metadata TEXT,
                    created_at TEXT DEFAULT CURRENT_TIMESTAMP
                )
            ''')
            await db.commit()
    
    async def bulk_insert_users(self, users: List[Dict[str, Any]]):
        """複数ユーザーの一括挿入"""
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("BEGIN TRANSACTION")
            
            try:
                await db.executemany(
                    "INSERT INTO users (username, email, metadata) VALUES (?, ?, ?)",
                    [(u['username'], u['email'], json.dumps(u.get('metadata', {}))) 
                     for u in users]
                )
                await db.commit()
                self.logger.info(f"Inserted {len(users)} users")
            except Exception as e:
                await db.rollback()
                self.logger.error(f"Bulk insert failed: {e}")
                raise
    
    async def get_users_with_pagination(self, page: int = 1, per_page: int = 10):
        """ページネーション付きでユーザーを取得"""
        offset = (page - 1) * per_page
        
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            
            # 総件数を取得
            cursor = await db.execute("SELECT COUNT(*) as total FROM users")
            total_row = await cursor.fetchone()
            total = total_row['total']
            
            # ページのデータを取得
            cursor = await db.execute(
                "SELECT * FROM users ORDER BY created_at DESC LIMIT ? OFFSET ?",
                (per_page, offset)
            )
            users = await cursor.fetchall()
            
            return {
                'users': [dict(user) for user in users],
                'page': page,
                'per_page': per_page,
                'total': total,
                'total_pages': (total + per_page - 1) // per_page
            }

# 非同期処理の実行例
async def async_example():
    db = AsyncUserDatabase('async_users.db')
    await db.initialize()
    
    # 一括挿入
    users = [
        {'username': f'user{i}', 'email': f'user{i}@example.com', 'metadata': {'age': 20 + i}}
        for i in range(100)
    ]
    await db.bulk_insert_users(users)
    
    # ページネーション取得
    result = await db.get_users_with_pagination(page=2, per_page=20)
    print(f"Page 2 users: {len(result['users'])}")
    print(f"Total pages: {result['total_pages']}")

# asyncio.run(async_example())
```

### トランザクション管理

```python
class TransactionManager:
    """トランザクションを管理するクラス"""
    
    def __init__(self, db_path):
        self.db_path = db_path
    
    def transfer_funds(self, from_account_id: int, to_account_id: int, amount: float):
        """資金移動のトランザクション"""
        conn = sqlite3.connect(self.db_path)
        
        try:
            # トランザクション開始
            conn.execute("BEGIN EXCLUSIVE")
            
            # 送金元の残高確認
            cursor = conn.execute(
                "SELECT balance FROM accounts WHERE id = ?", 
                (from_account_id,)
            )
            from_balance = cursor.fetchone()
            
            if not from_balance or from_balance[0] < amount:
                raise ValueError("Insufficient balance")
            
            # 送金処理
            conn.execute(
                "UPDATE accounts SET balance = balance - ? WHERE id = ?",
                (amount, from_account_id)
            )
            conn.execute(
                "UPDATE accounts SET balance = balance + ? WHERE id = ?",
                (amount, to_account_id)
            )
            
            # トランザクションログ
            conn.execute(
                '''INSERT INTO transaction_log 
                   (from_account_id, to_account_id, amount, status) 
                   VALUES (?, ?, ?, 'completed')''',
                (from_account_id, to_account_id, amount)
            )
            
            # コミット
            conn.commit()
            return True
            
        except Exception as e:
            # ロールバック
            conn.rollback()
            
            # エラーログ
            try:
                conn.execute(
                    '''INSERT INTO transaction_log 
                       (from_account_id, to_account_id, amount, status, error_message) 
                       VALUES (?, ?, ?, 'failed', ?)''',
                    (from_account_id, to_account_id, amount, str(e))
                )
                conn.commit()
            except:
                pass
            
            raise
        finally:
            conn.close()
```

## 8.3 JavaScript (Node.js) での SQLite 使用

### better-sqlite3を使用した実装

```javascript
const Database = require('better-sqlite3');
const path = require('path');

class UserRepository {
    constructor(dbPath) {
        this.db = new Database(dbPath);
        this.db.pragma('journal_mode = WAL');
        this.db.pragma('synchronous = NORMAL');
        this.initializeTables();
    }
    
    initializeTables() {
        // テーブル作成
        this.db.exec(`
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                username TEXT UNIQUE NOT NULL,
                email TEXT UNIQUE NOT NULL,
                profile TEXT,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
        `);
        
        // プリペアドステートメントの準備
        this.statements = {
            insertUser: this.db.prepare(
                'INSERT INTO users (username, email, profile) VALUES (?, ?, ?)'
            ),
            getUserById: this.db.prepare(
                'SELECT * FROM users WHERE id = ?'
            ),
            getUserByEmail: this.db.prepare(
                'SELECT * FROM users WHERE email = ?'
            ),
            updateUser: this.db.prepare(
                'UPDATE users SET username = ?, profile = ? WHERE id = ?'
            ),
            deleteUser: this.db.prepare(
                'DELETE FROM users WHERE id = ?'
            ),
            searchUsers: this.db.prepare(
                'SELECT * FROM users WHERE username LIKE ? OR email LIKE ? LIMIT ?'
            )
        };
    }
    
    createUser(username, email, profile = {}) {
        try {
            const result = this.statements.insertUser.run(
                username, 
                email, 
                JSON.stringify(profile)
            );
            return { id: result.lastInsertRowid, username, email, profile };
        } catch (error) {
            if (error.code === 'SQLITE_CONSTRAINT_UNIQUE') {
                throw new Error('Username or email already exists');
            }
            throw error;
        }
    }
    
    getUserById(id) {
        const user = this.statements.getUserById.get(id);
        if (user) {
            user.profile = JSON.parse(user.profile || '{}');
        }
        return user;
    }
    
    getUserByEmail(email) {
        const user = this.statements.getUserByEmail.get(email);
        if (user) {
            user.profile = JSON.parse(user.profile || '{}');
        }
        return user;
    }
    
    updateUser(id, updates) {
        const user = this.getUserById(id);
        if (!user) {
            throw new Error('User not found');
        }
        
        const username = updates.username || user.username;
        const profile = updates.profile || user.profile;
        
        const result = this.statements.updateUser.run(
            username,
            JSON.stringify(profile),
            id
        );
        
        return result.changes > 0;
    }
    
    searchUsers(query, limit = 10) {
        const searchPattern = `%${query}%`;
        const users = this.statements.searchUsers.all(
            searchPattern,
            searchPattern,
            limit
        );
        
        return users.map(user => {
            user.profile = JSON.parse(user.profile || '{}');
            return user;
        });
    }
    
    // トランザクション処理
    bulkCreateUsers(users) {
        const insert = this.db.prepare(
            'INSERT INTO users (username, email, profile) VALUES (?, ?, ?)'
        );
        
        const createMany = this.db.transaction((users) => {
            for (const user of users) {
                insert.run(user.username, user.email, JSON.stringify(user.profile || {}));
            }
            return users.length;
        });
        
        try {
            const count = createMany(users);
            return { success: true, count };
        } catch (error) {
            return { success: false, error: error.message };
        }
    }
    
    close() {
        this.db.close();
    }
}

// 使用例
const db = new UserRepository('./users.db');

try {
    // ユーザー作成
    const newUser = db.createUser('john_doe', 'john@example.com', {
        fullName: 'John Doe',
        age: 30
    });
    console.log('Created user:', newUser);
    
    // ユーザー検索
    const user = db.getUserByEmail('john@example.com');
    console.log('Found user:', user);
    
    // 一括作成
    const users = [
        { username: 'alice', email: 'alice@example.com', profile: { age: 25 } },
        { username: 'bob', email: 'bob@example.com', profile: { age: 35 } }
    ];
    const result = db.bulkCreateUsers(users);
    console.log('Bulk create result:', result);
    
} catch (error) {
    console.error('Error:', error.message);
} finally {
    db.close();
}
```

### 非同期処理とプロミス

```javascript
const sqlite3 = require('sqlite3').verbose();
const { promisify } = require('util');

class AsyncDatabase {
    constructor(dbPath) {
        this.db = new sqlite3.Database(dbPath);
        
        // メソッドをプロミス化
        this.run = promisify(this.db.run.bind(this.db));
        this.get = promisify(this.db.get.bind(this.db));
        this.all = promisify(this.db.all.bind(this.db));
    }
    
    async initialize() {
        await this.run(`
            CREATE TABLE IF NOT EXISTS tasks (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                description TEXT,
                status TEXT DEFAULT 'pending',
                priority INTEGER DEFAULT 0,
                due_date TEXT,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP,
                updated_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
        `);
        
        await this.run(`
            CREATE INDEX IF NOT EXISTS idx_tasks_status 
            ON tasks(status)
        `);
        
        await this.run(`
            CREATE INDEX IF NOT EXISTS idx_tasks_due_date 
            ON tasks(due_date)
        `);
    }
    
    async createTask(task) {
        const { title, description, priority, due_date } = task;
        const result = await this.run(
            `INSERT INTO tasks (title, description, priority, due_date) 
             VALUES (?, ?, ?, ?)`,
            [title, description, priority || 0, due_date]
        );
        return result.lastID;
    }
    
    async getTask(id) {
        return await this.get('SELECT * FROM tasks WHERE id = ?', [id]);
    }
    
    async updateTaskStatus(id, status) {
        const validStatuses = ['pending', 'in_progress', 'completed', 'cancelled'];
        if (!validStatuses.includes(status)) {
            throw new Error('Invalid status');
        }
        
        await this.run(
            `UPDATE tasks 
             SET status = ?, updated_at = CURRENT_TIMESTAMP 
             WHERE id = ?`,
            [status, id]
        );
    }
    
    async getTasksByStatus(status) {
        return await this.all(
            'SELECT * FROM tasks WHERE status = ? ORDER BY priority DESC, due_date ASC',
            [status]
        );
    }
    
    async getOverdueTasks() {
        return await this.all(
            `SELECT * FROM tasks 
             WHERE status != 'completed' 
               AND status != 'cancelled'
               AND due_date < date('now')
             ORDER BY due_date ASC`
        );
    }
    
    async getTaskStatistics() {
        const stats = await this.all(`
            SELECT 
                status,
                COUNT(*) as count,
                AVG(priority) as avg_priority
            FROM tasks
            GROUP BY status
        `);
        
        const total = await this.get('SELECT COUNT(*) as total FROM tasks');
        
        return {
            total: total.total,
            byStatus: stats
        };
    }
    
    close() {
        return new Promise((resolve, reject) => {
            this.db.close((err) => {
                if (err) reject(err);
                else resolve();
            });
        });
    }
}

// 使用例
async function taskManagerExample() {
    const db = new AsyncDatabase('./tasks.db');
    
    try {
        await db.initialize();
        
        // タスク作成
        const taskId = await db.createTask({
            title: '重要なプレゼンテーション',
            description: '四半期レビューのプレゼン資料作成',
            priority: 5,
            due_date: '2024-12-31'
        });
        console.log('Created task with ID:', taskId);
        
        // タスク取得
        const task = await db.getTask(taskId);
        console.log('Task details:', task);
        
        // ステータス更新
        await db.updateTaskStatus(taskId, 'in_progress');
        
        // 統計情報
        const stats = await db.getTaskStatistics();
        console.log('Task statistics:', stats);
        
    } catch (error) {
        console.error('Error:', error);
    } finally {
        await db.close();
    }
}

// taskManagerExample();
```

## 8.4 Java での SQLite 使用

### JDBC を使用した実装

```java
import java.sql.*;
import java.util.*;
import java.time.LocalDateTime;

public class SQLiteUserRepository {
    private final String dbUrl;
    
    public SQLiteUserRepository(String dbPath) {
        this.dbUrl = "jdbc:sqlite:" + dbPath;
        initializeDatabase();
    }
    
    private void initializeDatabase() {
        String createTableSQL = """
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                username TEXT UNIQUE NOT NULL,
                email TEXT UNIQUE NOT NULL,
                password_hash TEXT NOT NULL,
                is_active BOOLEAN DEFAULT 1,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP,
                updated_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
        """;
        
        try (Connection conn = DriverManager.getConnection(dbUrl);
             Statement stmt = conn.createStatement()) {
            stmt.execute(createTableSQL);
            
            // インデックス作成
            stmt.execute("CREATE INDEX IF NOT EXISTS idx_users_email ON users(email)");
            stmt.execute("CREATE INDEX IF NOT EXISTS idx_users_username ON users(username)");
        } catch (SQLException e) {
            throw new RuntimeException("Failed to initialize database", e);
        }
    }
    
    public class User {
        private Integer id;
        private String username;
        private String email;
        private String passwordHash;
        private Boolean isActive;
        private LocalDateTime createdAt;
        private LocalDateTime updatedAt;
        
        // コンストラクタ、ゲッター、セッター省略
    }
    
    public User createUser(String username, String email, String passwordHash) {
        String sql = "INSERT INTO users (username, email, password_hash) VALUES (?, ?, ?)";
        
        try (Connection conn = DriverManager.getConnection(dbUrl);
             PreparedStatement pstmt = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            
            pstmt.setString(1, username);
            pstmt.setString(2, email);
            pstmt.setString(3, passwordHash);
            
            int affectedRows = pstmt.executeUpdate();
            
            if (affectedRows == 0) {
                throw new SQLException("Creating user failed, no rows affected.");
            }
            
            try (ResultSet generatedKeys = pstmt.getGeneratedKeys()) {
                if (generatedKeys.next()) {
                    User user = new User();
                    user.setId(generatedKeys.getInt(1));
                    user.setUsername(username);
                    user.setEmail(email);
                    user.setPasswordHash(passwordHash);
                    user.setIsActive(true);
                    return user;
                } else {
                    throw new SQLException("Creating user failed, no ID obtained.");
                }
            }
        } catch (SQLException e) {
            throw new RuntimeException("Failed to create user", e);
        }
    }
    
    public Optional<User> findUserByEmail(String email) {
        String sql = "SELECT * FROM users WHERE email = ?";
        
        try (Connection conn = DriverManager.getConnection(dbUrl);
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            pstmt.setString(1, email);
            
            try (ResultSet rs = pstmt.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(mapResultSetToUser(rs));
                }
            }
        } catch (SQLException e) {
            throw new RuntimeException("Failed to find user", e);
        }
        
        return Optional.empty();
    }
    
    public List<User> searchUsers(String query, int limit) {
        String sql = "SELECT * FROM users WHERE username LIKE ? OR email LIKE ? LIMIT ?";
        List<User> users = new ArrayList<>();
        
        try (Connection conn = DriverManager.getConnection(dbUrl);
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            String searchPattern = "%" + query + "%";
            pstmt.setString(1, searchPattern);
            pstmt.setString(2, searchPattern);
            pstmt.setInt(3, limit);
            
            try (ResultSet rs = pstmt.executeQuery()) {
                while (rs.next()) {
                    users.add(mapResultSetToUser(rs));
                }
            }
        } catch (SQLException e) {
            throw new RuntimeException("Failed to search users", e);
        }
        
        return users;
    }
    
    // トランザクション処理
    public void transferData(int fromUserId, int toUserId, Map<String, Object> data) {
        Connection conn = null;
        
        try {
            conn = DriverManager.getConnection(dbUrl);
            conn.setAutoCommit(false);  // トランザクション開始
            
            // データの取得と検証
            String selectSQL = "SELECT * FROM users WHERE id = ? FOR UPDATE";
            PreparedStatement selectStmt = conn.prepareStatement(selectSQL);
            selectStmt.setInt(1, fromUserId);
            ResultSet rs = selectStmt.executeQuery();
            
            if (!rs.next()) {
                throw new SQLException("Source user not found");
            }
            
            // データの転送処理
            String updateSQL = "UPDATE users SET metadata = ? WHERE id = ?";
            PreparedStatement updateStmt = conn.prepareStatement(updateSQL);
            updateStmt.setString(1, data.toString());
            updateStmt.setInt(2, toUserId);
            updateStmt.executeUpdate();
            
            // ログ記録
            String logSQL = "INSERT INTO transfer_log (from_user_id, to_user_id, data) VALUES (?, ?, ?)";
            PreparedStatement logStmt = conn.prepareStatement(logSQL);
            logStmt.setInt(1, fromUserId);
            logStmt.setInt(2, toUserId);
            logStmt.setString(3, data.toString());
            logStmt.executeUpdate();
            
            conn.commit();  // トランザクションコミット
            
        } catch (SQLException e) {
            if (conn != null) {
                try {
                    conn.rollback();  // ロールバック
                } catch (SQLException ex) {
                    ex.printStackTrace();
                }
            }
            throw new RuntimeException("Transfer failed", e);
        } finally {
            if (conn != null) {
                try {
                    conn.setAutoCommit(true);
                    conn.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    
    private User mapResultSetToUser(ResultSet rs) throws SQLException {
        User user = new User();
        user.setId(rs.getInt("id"));
        user.setUsername(rs.getString("username"));
        user.setEmail(rs.getString("email"));
        user.setPasswordHash(rs.getString("password_hash"));
        user.setIsActive(rs.getBoolean("is_active"));
        // 日時の変換は省略
        return user;
    }
}
```

### コネクションプールの実装

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

public class SQLiteConnectionPool {
    private static HikariDataSource dataSource;
    
    static {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:sqlite:application.db");
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(2);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        
        // SQLite特有の設定
        config.addDataSourceProperty("journal_mode", "WAL");
        config.addDataSourceProperty("synchronous", "NORMAL");
        config.addDataSourceProperty("temp_store", "MEMORY");
        config.addDataSourceProperty("cache_size", "-64000");
        
        dataSource = new HikariDataSource(config);
    }
    
    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
    
    public static void close() {
        if (dataSource != null && !dataSource.isClosed()) {
            dataSource.close();
        }
    }
}
```

## 8.5 C# での SQLite 使用

```csharp
using System;
using System.Collections.Generic;
using System.Data.SQLite;
using System.Threading.Tasks;
using Dapper;

public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string Email { get; set; }
    public string PasswordHash { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}

public class SQLiteUserRepository : IDisposable
{
    private readonly string _connectionString;
    
    public SQLiteUserRepository(string dbPath)
    {
        _connectionString = $"Data Source={dbPath};Version=3;";
        InitializeDatabase();
    }
    
    private void InitializeDatabase()
    {
        using var connection = new SQLiteConnection(_connectionString);
        connection.Open();
        
        var createTableCommand = @"
            CREATE TABLE IF NOT EXISTS users (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Username TEXT UNIQUE NOT NULL,
                Email TEXT UNIQUE NOT NULL,
                PasswordHash TEXT NOT NULL,
                IsActive BOOLEAN DEFAULT 1,
                CreatedAt TEXT DEFAULT CURRENT_TIMESTAMP,
                UpdatedAt TEXT DEFAULT CURRENT_TIMESTAMP
            )";
        
        connection.Execute(createTableCommand);
        connection.Execute("CREATE INDEX IF NOT EXISTS idx_users_email ON users(Email)");
    }
    
    public async Task<User> CreateUserAsync(string username, string email, string passwordHash)
    {
        using var connection = new SQLiteConnection(_connectionString);
        
        var sql = @"
            INSERT INTO users (Username, Email, PasswordHash) 
            VALUES (@Username, @Email, @PasswordHash);
            SELECT last_insert_rowid();";
        
        var id = await connection.QuerySingleAsync<int>(sql, new
        {
            Username = username,
            Email = email,
            PasswordHash = passwordHash
        });
        
        return new User
        {
            Id = id,
            Username = username,
            Email = email,
            PasswordHash = passwordHash,
            IsActive = true,
            CreatedAt = DateTime.Now,
            UpdatedAt = DateTime.Now
        };
    }
    
    public async Task<User> GetUserByEmailAsync(string email)
    {
        using var connection = new SQLiteConnection(_connectionString);
        
        var sql = "SELECT * FROM users WHERE Email = @Email";
        return await connection.QuerySingleOrDefaultAsync<User>(sql, new { Email = email });
    }
    
    public async Task<IEnumerable<User>> SearchUsersAsync(string query, int limit = 10)
    {
        using var connection = new SQLiteConnection(_connectionString);
        
        var sql = @"
            SELECT * FROM users 
            WHERE Username LIKE @Query OR Email LIKE @Query 
            LIMIT @Limit";
        
        return await connection.QueryAsync<User>(sql, new
        {
            Query = $"%{query}%",
            Limit = limit
        });
    }
    
    // トランザクション処理
    public async Task<bool> BulkUpdateUsersAsync(List<User> users)
    {
        using var connection = new SQLiteConnection(_connectionString);
        connection.Open();
        
        using var transaction = connection.BeginTransaction();
        
        try
        {
            var sql = @"
                UPDATE users 
                SET Username = @Username, 
                    Email = @Email, 
                    IsActive = @IsActive,
                    UpdatedAt = CURRENT_TIMESTAMP
                WHERE Id = @Id";
            
            foreach (var user in users)
            {
                await connection.ExecuteAsync(sql, user, transaction);
            }
            
            transaction.Commit();
            return true;
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }
    
    // 非同期バッチ処理
    public async Task<int> BatchInsertUsersAsync(IEnumerable<User> users)
    {
        using var connection = new SQLiteConnection(_connectionString);
        connection.Open();
        
        // パフォーマンス設定
        await connection.ExecuteAsync("PRAGMA synchronous = OFF");
        await connection.ExecuteAsync("PRAGMA journal_mode = MEMORY");
        
        using var transaction = connection.BeginTransaction();
        
        try
        {
            var sql = @"
                INSERT INTO users (Username, Email, PasswordHash, IsActive) 
                VALUES (@Username, @Email, @PasswordHash, @IsActive)";
            
            var count = await connection.ExecuteAsync(sql, users, transaction);
            
            transaction.Commit();
            
            // 設定を元に戻す
            await connection.ExecuteAsync("PRAGMA synchronous = FULL");
            await connection.ExecuteAsync("PRAGMA journal_mode = DELETE");
            
            return count;
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }
    
    public void Dispose()
    {
        SQLiteConnection.ClearAllPools();
    }
}

// 使用例
class Program
{
    static async Task Main(string[] args)
    {
        using var repository = new SQLiteUserRepository("users.db");
        
        // ユーザー作成
        var newUser = await repository.CreateUserAsync(
            "john_doe", 
            "john@example.com", 
            "hashed_password"
        );
        Console.WriteLine($"Created user: {newUser.Id}");
        
        // ユーザー検索
        var users = await repository.SearchUsersAsync("john");
        foreach (var user in users)
        {
            Console.WriteLine($"Found: {user.Username} - {user.Email}");
        }
    }
}
```

## 8.6 ORM（Object-Relational Mapping）の使用

### Python - SQLAlchemy

```python
from sqlalchemy import create_engine, Column, Integer, String, Boolean, DateTime, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, relationship, scoped_session
from sqlalchemy.sql import func
from contextlib import contextmanager
import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True)
    username = Column(String(50), unique=True, nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())
    
    # リレーションシップ
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")
    
    def __repr__(self):
        return f"<User(username='{self.username}', email='{self.email}')>"

class Post(Base):
    __tablename__ = 'posts'
    
    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    content = Column(String, nullable=False)
    author_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    created_at = Column(DateTime, default=func.now())
    
    # リレーションシップ
    author = relationship("User", back_populates="posts")
    tags = relationship("Tag", secondary="post_tags", back_populates="posts")

class Tag(Base):
    __tablename__ = 'tags'
    
    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True, nullable=False)
    
    # リレーションシップ
    posts = relationship("Post", secondary="post_tags", back_populates="tags")

# 中間テーブル
post_tags = Table('post_tags', Base.metadata,
    Column('post_id', Integer, ForeignKey('posts.id'), primary_key=True),
    Column('tag_id', Integer, ForeignKey('tags.id'), primary_key=True)
)

class DatabaseManager:
    def __init__(self, db_url='sqlite:///blog.db'):
        self.engine = create_engine(db_url, echo=False)
        Base.metadata.create_all(self.engine)
        self.SessionLocal = sessionmaker(bind=self.engine)
        self.Session = scoped_session(self.SessionLocal)
    
    @contextmanager
    def session_scope(self):
        """セッションのコンテキストマネージャー"""
        session = self.Session()
        try:
            yield session
            session.commit()
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()
    
    def create_user(self, username, email):
        with self.session_scope() as session:
            user = User(username=username, email=email)
            session.add(user)
            return user.id
    
    def create_post(self, author_id, title, content, tag_names=None):
        with self.session_scope() as session:
            post = Post(author_id=author_id, title=title, content=content)
            
            if tag_names:
                for tag_name in tag_names:
                    tag = session.query(Tag).filter_by(name=tag_name).first()
                    if not tag:
                        tag = Tag(name=tag_name)
                    post.tags.append(tag)
            
            session.add(post)
            return post.id
    
    def get_user_with_posts(self, user_id):
        with self.session_scope() as session:
            user = session.query(User).filter_by(id=user_id).first()
            if user:
                # 遅延読み込みを回避
                posts = user.posts
                return {
                    'user': user,
                    'posts': posts,
                    'post_count': len(posts)
                }
            return None
    
    def search_posts(self, keyword):
        with self.session_scope() as session:
            posts = session.query(Post).filter(
                Post.title.contains(keyword) | Post.content.contains(keyword)
            ).all()
            return posts

# 使用例
db = DatabaseManager()

# ユーザー作成
user_id = db.create_user('alice', 'alice@example.com')

# 投稿作成
post_id = db.create_post(
    author_id=user_id,
    title='SQLiteの使い方',
    content='SQLiteは素晴らしいデータベースです。',
    tag_names=['sqlite', 'database', 'tutorial']
)

# ユーザーと投稿を取得
user_data = db.get_user_with_posts(user_id)
print(f"User: {user_data['user'].username}")
print(f"Posts: {user_data['post_count']}")
```

### JavaScript - Sequelize

```javascript
const { Sequelize, DataTypes, Op } = require('sequelize');

// SQLiteデータベースの設定
const sequelize = new Sequelize({
    dialect: 'sqlite',
    storage: './blog.db',
    logging: false,
    define: {
        timestamps: true,
        underscored: true
    }
});

// モデル定義
const User = sequelize.define('User', {
    username: {
        type: DataTypes.STRING,
        allowNull: false,
        unique: true,
        validate: {
            len: [3, 50]
        }
    },
    email: {
        type: DataTypes.STRING,
        allowNull: false,
        unique: true,
        validate: {
            isEmail: true
        }
    },
    isActive: {
        type: DataTypes.BOOLEAN,
        defaultValue: true
    }
});

const Post = sequelize.define('Post', {
    title: {
        type: DataTypes.STRING,
        allowNull: false,
        validate: {
            len: [1, 200]
        }
    },
    content: {
        type: DataTypes.TEXT,
        allowNull: false
    },
    viewCount: {
        type: DataTypes.INTEGER,
        defaultValue: 0
    }
});

const Tag = sequelize.define('Tag', {
    name: {
        type: DataTypes.STRING,
        allowNull: false,
        unique: true
    }
});

// アソシエーション定義
User.hasMany(Post, { as: 'posts', foreignKey: 'authorId' });
Post.belongsTo(User, { as: 'author', foreignKey: 'authorId' });

Post.belongsToMany(Tag, { through: 'PostTags' });
Tag.belongsToMany(Post, { through: 'PostTags' });

// リポジトリクラス
class BlogRepository {
    async initialize() {
        await sequelize.sync();
    }
    
    async createUser(userData) {
        return await User.create(userData);
    }
    
    async createPost(authorId, postData, tagNames = []) {
        const transaction = await sequelize.transaction();
        
        try {
            // 投稿作成
            const post = await Post.create({
                ...postData,
                authorId
            }, { transaction });
            
            // タグ処理
            if (tagNames.length > 0) {
                const tags = await Promise.all(
                    tagNames.map(name => 
                        Tag.findOrCreate({
                            where: { name },
                            transaction
                        })
                    )
                );
                
                await post.setTags(tags.map(([tag]) => tag), { transaction });
            }
            
            await transaction.commit();
            return post;
            
        } catch (error) {
            await transaction.rollback();
            throw error;
        }
    }
    
    async getUserWithPosts(userId) {
        return await User.findByPk(userId, {
            include: [{
                model: Post,
                as: 'posts',
                include: [{
                    model: Tag,
                    through: { attributes: [] }
                }]
            }]
        });
    }
    
    async searchPosts(keyword, options = {}) {
        const { limit = 10, offset = 0 } = options;
        
        return await Post.findAndCountAll({
            where: {
                [Op.or]: [
                    { title: { [Op.like]: `%${keyword}%` } },
                    { content: { [Op.like]: `%${keyword}%` } }
                ]
            },
            include: [
                { model: User, as: 'author', attributes: ['id', 'username'] },
                { model: Tag, through: { attributes: [] } }
            ],
            limit,
            offset,
            order: [['createdAt', 'DESC']]
        });
    }
    
    async getPopularPosts(days = 7) {
        const dateThreshold = new Date();
        dateThreshold.setDate(dateThreshold.getDate() - days);
        
        return await Post.findAll({
            where: {
                createdAt: { [Op.gte]: dateThreshold }
            },
            order: [['viewCount', 'DESC']],
            limit: 10,
            include: [
                { model: User, as: 'author', attributes: ['username'] }
            ]
        });
    }
}

// 使用例
async function example() {
    const repo = new BlogRepository();
    await repo.initialize();
    
    try {
        // ユーザー作成
        const user = await repo.createUser({
            username: 'john_doe',
            email: 'john@example.com'
        });
        
        // 投稿作成
        const post = await repo.createPost(user.id, {
            title: 'Sequelizeの使い方',
            content: 'SequelizeでSQLiteを使う方法を解説します。'
        }, ['sequelize', 'sqlite', 'orm']);
        
        // 検索
        const searchResults = await repo.searchPosts('SQLite');
        console.log(`Found ${searchResults.count} posts`);
        
    } catch (error) {
        console.error('Error:', error);
    }
}

// example();
```

## 8.7 パフォーマンス最適化のテクニック

### プリペアドステートメントとバッチ処理

```python
import sqlite3
import time
from concurrent.futures import ThreadPoolExecutor
import threading

class OptimizedDatabase:
    def __init__(self, db_path):
        self.db_path = db_path
        self.local = threading.local()
        
    def get_connection(self):
        if not hasattr(self.local, 'connection'):
            self.local.connection = sqlite3.connect(self.db_path)
            self.local.connection.execute("PRAGMA journal_mode=WAL")
            self.local.connection.execute("PRAGMA synchronous=NORMAL")
            self.local.connection.execute("PRAGMA cache_size=-64000")
            self.local.connection.execute("PRAGMA temp_store=MEMORY")
        return self.local.connection
    
    def benchmark_insert_methods(self, num_records=10000):
        """異なる挿入方法のベンチマーク"""
        results = {}
        
        # 方法1: 個別INSERT（遅い）
        start = time.time()
        conn = self.get_connection()
        for i in range(1000):  # 少なめでテスト
            conn.execute(
                "INSERT INTO test_table (data) VALUES (?)",
                (f"data_{i}",)
            )
        conn.commit()
        results['individual_inserts'] = time.time() - start
        
        # 方法2: トランザクション内での個別INSERT
        start = time.time()
        conn.execute("BEGIN")
        for i in range(num_records):
            conn.execute(
                "INSERT INTO test_table (data) VALUES (?)",
                (f"data_{i}",)
            )
        conn.commit()
        results['transaction_inserts'] = time.time() - start
        
        # 方法3: executemany
        start = time.time()
        data = [(f"data_{i}",) for i in range(num_records)]
        conn.executemany("INSERT INTO test_table (data) VALUES (?)", data)
        conn.commit()
        results['executemany'] = time.time() - start
        
        return results
    
    def parallel_processing(self, tasks):
        """並列処理の例"""
        def process_task(task_data):
            conn = self.get_connection()
            cursor = conn.cursor()
            
            # 各タスクの処理
            cursor.execute("BEGIN")
            for item in task_data:
                cursor.execute(
                    "INSERT INTO processed_data (result) VALUES (?)",
                    (process_item(item),)
                )
            conn.commit()
        
        with ThreadPoolExecutor(max_workers=4) as executor:
            # タスクを分割して並列実行
            chunk_size = len(tasks) // 4
            chunks = [tasks[i:i + chunk_size] for i in range(0, len(tasks), chunk_size)]
            executor.map(process_task, chunks)

def process_item(item):
    # 何らかの処理
    return f"processed_{item}"
```

## 8.8 エラーハンドリングとロギング

```python
import logging
import functools
from enum import Enum

class DatabaseError(Exception):
    """データベース関連のカスタム例外"""
    pass

class TransactionError(DatabaseError):
    """トランザクション関連のエラー"""
    pass

class DataIntegrityError(DatabaseError):
    """データ整合性エラー"""
    pass

def with_database_error_handling(func):
    """データベースエラーハンドリングのデコレータ"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except sqlite3.IntegrityError as e:
            logging.error(f"Integrity error in {func.__name__}: {e}")
            raise DataIntegrityError(str(e))
        except sqlite3.OperationalError as e:
            logging.error(f"Operational error in {func.__name__}: {e}")
            raise DatabaseError(str(e))
        except Exception as e:
            logging.error(f"Unexpected error in {func.__name__}: {e}")
            raise
    return wrapper

class RobustDatabase:
    def __init__(self, db_path):
        self.db_path = db_path
        self.logger = logging.getLogger(__name__)
        
    @with_database_error_handling
    def safe_insert(self, table, data):
        """エラーハンドリング付きの安全な挿入"""
        conn = sqlite3.connect(self.db_path)
        try:
            columns = ', '.join(data.keys())
            placeholders = ', '.join(['?' for _ in data])
            sql = f"INSERT INTO {table} ({columns}) VALUES ({placeholders})"
            
            conn.execute(sql, list(data.values()))
            conn.commit()
            
            self.logger.info(f"Successfully inserted into {table}")
            return True
            
        except Exception as e:
            conn.rollback()
            self.logger.error(f"Failed to insert into {table}: {e}")
            raise
        finally:
            conn.close()
```

## 8.9 まとめ

この章では、様々なプログラミング言語からSQLiteを使用する方法を学びました。各言語には特有のライブラリやフレームワークがありますが、基本的な概念は共通しています。

### この章で学んだこと

- Python、JavaScript、Java、C#でのSQLite接続方法
- プリペアドステートメントの使用
- トランザクション管理の実装
- ORMを使用したデータベース操作
- エラーハンドリングとロギング
- パフォーマンス最適化のテクニック

### 重要なポイント

1. **接続管理**: データベース接続は適切に管理し、使用後は必ず閉じる
2. **プリペアドステートメント**: SQLインジェクション対策とパフォーマンス向上
3. **トランザクション**: データの整合性を保つため適切に使用
4. **エラーハンドリング**: 予期しないエラーに対する準備
5. **パフォーマンス**: バッチ処理、インデックス、適切な設定の活用

### 確認問題

1. プリペアドステートメントを使用する利点を2つ挙げてください
2. トランザクションを使用すべき場面の例を3つ挙げてください
3. ORMを使用する利点と欠点をそれぞれ説明してください
4. データベース接続プールが必要な理由を説明してください

---

**[← 第7章: トランザクションへ戻る](./chapter07-transactions.md)** | **[→ 第9章: 実践演習へ進む](./chapter09-practice.md)**