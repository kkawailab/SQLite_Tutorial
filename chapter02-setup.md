# 第2章: 環境構築

## 2.1 はじめに

この章では、SQLiteを実際に使用するための環境構築について説明します。SQLiteは非常にシンプルなデータベースエンジンなので、複雑なインストール作業は必要ありません。各オペレーティングシステムでのインストール方法と、便利なツールの導入方法を解説します。

## 2.2 SQLiteのインストール

### Windows

#### 方法1: 公式サイトからダウンロード

1. SQLite公式サイト（https://www.sqlite.org/download.html）にアクセス
2. "Precompiled Binaries for Windows"セクションから以下をダウンロード：
   - `sqlite-tools-win32-x86-XXXXXXX.zip`（コマンドラインツール）

3. ダウンロードしたZIPファイルを任意のフォルダに解凍（例：`C:\sqlite`）

4. 環境変数PATHに追加：
   ```powershell
   # PowerShellを管理者権限で実行
   [Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\sqlite", [EnvironmentVariableTarget]::Machine)
   ```

5. 新しいコマンドプロンプトを開いて確認：
   ```cmd
   sqlite3 --version
   ```

#### 方法2: Chocolateyを使用（推奨）

```powershell
# PowerShellを管理者権限で実行
choco install sqlite
```

### macOS

#### 方法1: Homebrewを使用（推奨）

```bash
# Homebrewがインストールされていない場合は先にインストール
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# SQLiteのインストール
brew install sqlite

# バージョン確認
sqlite3 --version
```

#### 方法2: macOS標準のSQLiteを使用

macOSには標準でSQLiteがインストールされています：

```bash
# バージョン確認
sqlite3 --version

# 通常は /usr/bin/sqlite3 に存在
which sqlite3
```

### Linux (Ubuntu/Debian)

```bash
# パッケージリストの更新
sudo apt update

# SQLite3のインストール
sudo apt install sqlite3

# 開発用ライブラリもインストール（プログラミングで使用する場合）
sudo apt install libsqlite3-dev

# バージョン確認
sqlite3 --version
```

### Linux (CentOS/RHEL/Fedora)

```bash
# SQLite3のインストール
sudo yum install sqlite

# または（Fedora 22以降）
sudo dnf install sqlite

# 開発用ライブラリ
sudo yum install sqlite-devel

# バージョン確認
sqlite3 --version
```

## 2.3 SQLiteコマンドラインツールの使い方

### 基本的な起動方法

```bash
# 新規データベースの作成または既存データベースを開く
sqlite3 mydatabase.db

# メモリ上の一時的なデータベースを作成
sqlite3 :memory:

# 読み取り専用モードで開く
sqlite3 -readonly mydatabase.db
```

### SQLiteシェルの基本コマンド

SQLiteシェル内で使用できる特別なコマンド（ドットコマンド）：

```sql
-- ヘルプの表示
.help

-- データベース一覧の表示
.databases

-- テーブル一覧の表示
.tables

-- テーブルのスキーマ表示
.schema table_name

-- 出力モードの変更
.mode column    -- カラム形式
.mode csv       -- CSV形式
.mode json      -- JSON形式
.mode table     -- テーブル形式（見やすい）

-- ヘッダーの表示/非表示
.headers on
.headers off

-- 終了
.exit または .quit
```

### 実践例：最初のデータベース作成

```bash
# データベースの作成
sqlite3 test.db

# SQLiteシェル内での操作
sqlite> CREATE TABLE users (
   ...>     id INTEGER PRIMARY KEY,
   ...>     name TEXT NOT NULL,
   ...>     email TEXT UNIQUE
   ...> );

sqlite> INSERT INTO users (name, email) VALUES ('山田太郎', 'yamada@example.com');

sqlite> SELECT * FROM users;
1|山田太郎|yamada@example.com

sqlite> .exit
```

## 2.4 GUI ツールの紹介

### DB Browser for SQLite（推奨）

無料でオープンソースの、最も人気のあるSQLite GUIツールです。

#### インストール方法

**Windows/macOS/Linux:**
1. 公式サイト（https://sqlitebrowser.org/）からダウンロード
2. インストーラーを実行

**macOS (Homebrew):**
```bash
brew install --cask db-browser-for-sqlite
```

**Linux (Ubuntu):**
```bash
sudo apt install sqlitebrowser
```

#### 主な機能
- データベースの作成・編集
- テーブルの視覚的な作成・変更
- データの閲覧・編集
- SQLクエリの実行
- インポート/エクスポート機能

### SQLiteStudio

もう一つの人気のある無料GUIツールです。

#### 特徴
- ポータブル版あり（インストール不要）
- プラグインシステム
- 高度なSQL編集機能
- 複数のデータベースを同時に開ける

### TablePlus（有料/無料版あり）

モダンなUIを持つデータベース管理ツールです。

#### 特徴
- 美しいUI
- 高速な動作
- 複数のデータベースタイプに対応
- ダークモード対応

## 2.5 VSCodeでの開発環境

Visual Studio Codeを使用したSQLite開発環境の構築：

### 拡張機能のインストール

1. **SQLite Viewer**
   - SQLiteデータベースファイルを直接VSCode内で閲覧
   - 拡張機能IDを検索: `qwtel.sqlite-viewer`

2. **SQLite**
   - SQLクエリの実行とデータベース管理
   - 拡張機能ID: `alexcvzz.vscode-sqlite`

### 設定例

`.vscode/settings.json`:
```json
{
    "sqlite.logLevel": "INFO",
    "sqlite.database": {
        "default": "${workspaceFolder}/mydatabase.db"
    }
}
```

## 2.6 プログラミング言語用ライブラリ

### Python

```bash
# sqlite3は標準ライブラリに含まれているため、追加インストール不要
python -c "import sqlite3; print(sqlite3.version)"
```

### Node.js

```bash
# better-sqlite3（推奨）
npm install better-sqlite3

# node-sqlite3
npm install sqlite3
```

### Java

```xml
<!-- Maven -->
<dependency>
    <groupId>org.xerial</groupId>
    <artifactId>sqlite-jdbc</artifactId>
    <version>3.36.0.3</version>
</dependency>
```

### C/C++

```bash
# Linux/macOS
gcc myprogram.c -lsqlite3 -o myprogram

# ヘッダーファイルの確認
ls /usr/include/sqlite3.h
```

## 2.7 動作確認

環境構築が正しく完了したか確認しましょう。

### コマンドラインでの確認

```bash
# SQLiteバージョンの確認
sqlite3 --version

# 簡単なテスト
echo "SELECT sqlite_version();" | sqlite3

# メモリデータベースでのテスト
sqlite3 :memory: "SELECT datetime('now', 'localtime');"
```

### Pythonでの確認

```python
import sqlite3

# バージョン情報の表示
print(f"SQLite version: {sqlite3.sqlite_version}")
print(f"Python sqlite3 module version: {sqlite3.version}")

# 簡単なテスト
conn = sqlite3.connect(':memory:')
cursor = conn.cursor()
cursor.execute("SELECT datetime('now', 'localtime')")
print(f"Current time: {cursor.fetchone()[0]}")
conn.close()
```

## 2.8 トラブルシューティング

### よくある問題と解決方法

#### 1. "sqlite3: command not found"

**原因**: PATHが正しく設定されていない
**解決方法**:
```bash
# 実行ファイルの場所を確認
find / -name sqlite3 2>/dev/null

# PATHに追加（.bashrcや.zshrcに追記）
export PATH=$PATH:/path/to/sqlite3
```

#### 2. "database is locked"

**原因**: 他のプロセスがデータベースを使用中
**解決方法**:
- 他のSQLiteプロセスを終了
- データベースファイルの権限を確認
- ジャーナルファイル（.db-journal）を削除

#### 3. 文字化けが発生する

**原因**: 文字エンコーディングの問題
**解決方法**:
```sql
-- UTF-8エンコーディングを確認
PRAGMA encoding;

-- UTF-8に設定（新規データベースのみ）
PRAGMA encoding = "UTF-8";
```

## 2.9 推奨される開発環境

### 初心者向け

1. **DB Browser for SQLite** - 視覚的な操作
2. **SQLite3コマンドライン** - 基本的なSQL学習
3. **VSCode + SQLite拡張** - コード記述

### 中級者以上向け

1. **コマンドラインツール** - 効率的な操作
2. **お好みのエディタ + SQLite拡張** - 開発効率
3. **プログラミング言語のライブラリ** - アプリケーション開発

## 2.10 まとめ

この章では、SQLiteの環境構築について学びました。SQLiteは他のデータベースシステムと比較して、非常に簡単にセットアップできることが分かったと思います。

### この章で学んだこと

- 各OSでのSQLiteのインストール方法
- コマンドラインツールの基本的な使い方
- GUI ツールの種類と特徴
- 開発環境の構築方法
- トラブルシューティングの基本

### 次のステップ

環境構築が完了したので、次の章では実際にSQLiteでデータベースを作成し、基本的なSQL操作を学んでいきます。

### 確認課題

1. SQLiteが正しくインストールされているか、バージョンを確認してください
2. 任意のGUIツールをインストールし、起動できることを確認してください
3. テスト用のデータベースファイルを作成し、簡単なテーブルを作成してみてください
4. 作成したデータベースファイルのサイズを確認してください

---

**[← 第1章: SQLite入門へ戻る](./chapter01-introduction.md)** | **[→ 第3章: 基本的なSQL操作へ進む](./chapter03-basic-sql.md)**