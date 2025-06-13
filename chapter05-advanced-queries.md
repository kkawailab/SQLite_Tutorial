# 第5章: 高度なクエリ

## 5.1 はじめに

この章では、SQLiteでより複雑で高度なクエリを作成する方法を学びます。複数のテーブルを結合するJOIN操作、データを集約するGROUP BY、サブクエリ、ビューの作成など、実務で必要となる高度なSQL技術を習得します。

## 5.2 テーブルの結合（JOIN）

### サンプルデータの準備

まず、JOIN操作を学ぶためのサンプルテーブルを作成します。

```sql
-- 従業員テーブル
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    department_id INTEGER,
    salary REAL,
    hire_date TEXT
);

-- 部署テーブル
CREATE TABLE departments (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    location TEXT
);

-- プロジェクトテーブル
CREATE TABLE projects (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    start_date TEXT,
    end_date TEXT,
    budget REAL
);

-- 従業員とプロジェクトの関連テーブル
CREATE TABLE employee_projects (
    employee_id INTEGER,
    project_id INTEGER,
    role TEXT,
    PRIMARY KEY (employee_id, project_id),
    FOREIGN KEY (employee_id) REFERENCES employees(id),
    FOREIGN KEY (project_id) REFERENCES projects(id)
);

-- サンプルデータの挿入
INSERT INTO departments (id, name, location) VALUES
    (1, '開発部', '東京'),
    (2, '営業部', '大阪'),
    (3, '人事部', '東京'),
    (4, '経理部', '福岡');

INSERT INTO employees (id, name, department_id, salary, hire_date) VALUES
    (1, '山田太郎', 1, 500000, '2020-04-01'),
    (2, '鈴木花子', 1, 450000, '2021-04-01'),
    (3, '田中一郎', 2, 480000, '2019-04-01'),
    (4, '佐藤美咲', 3, 420000, '2022-04-01'),
    (5, '高橋健', NULL, 400000, '2023-04-01'); -- 部署未配属

INSERT INTO projects (id, name, start_date, end_date, budget) VALUES
    (1, 'Webサイトリニューアル', '2024-01-01', '2024-06-30', 5000000),
    (2, '新商品開発', '2024-03-01', '2024-12-31', 8000000),
    (3, '社内システム改善', '2024-02-01', '2024-08-31', 3000000);

INSERT INTO employee_projects (employee_id, project_id, role) VALUES
    (1, 1, 'プロジェクトリーダー'),
    (2, 1, '開発者'),
    (1, 2, '技術アドバイザー'),
    (3, 2, '営業担当'),
    (2, 3, '開発者'),
    (4, 3, '人事担当');
```

### INNER JOIN

INNER JOINは、両方のテーブルに一致するレコードのみを返します。

```sql
-- 基本的なINNER JOIN
SELECT 
    e.name AS 従業員名,
    d.name AS 部署名
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;

-- 複数条件でのJOIN
SELECT 
    e.name AS 従業員名,
    d.name AS 部署名,
    e.salary AS 給与
FROM employees e
INNER JOIN departments d ON e.department_id = d.id
WHERE d.location = '東京' AND e.salary > 450000;

-- 3つのテーブルをJOIN
SELECT 
    e.name AS 従業員名,
    d.name AS 部署名,
    p.name AS プロジェクト名,
    ep.role AS 役割
FROM employees e
INNER JOIN departments d ON e.department_id = d.id
INNER JOIN employee_projects ep ON e.id = ep.employee_id
INNER JOIN projects p ON ep.project_id = p.id
ORDER BY e.name, p.name;
```

### LEFT JOIN (LEFT OUTER JOIN)

LEFT JOINは、左側のテーブルのすべてのレコードと、右側のテーブルの一致するレコードを返します。

```sql
-- 部署未配属の従業員も含めて表示
SELECT 
    e.name AS 従業員名,
    d.name AS 部署名
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;

-- プロジェクトに参加していない従業員も表示
SELECT 
    e.name AS 従業員名,
    COUNT(ep.project_id) AS 参加プロジェクト数
FROM employees e
LEFT JOIN employee_projects ep ON e.id = ep.employee_id
GROUP BY e.id, e.name
ORDER BY 参加プロジェクト数 DESC;

-- 部署ごとの従業員数（従業員のいない部署も表示）
SELECT 
    d.name AS 部署名,
    d.location AS 所在地,
    COUNT(e.id) AS 従業員数
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name, d.location
ORDER BY 従業員数 DESC;
```

### CROSS JOIN

CROSS JOINは、2つのテーブルのデカルト積（すべての組み合わせ）を返します。

```sql
-- すべての従業員とプロジェクトの組み合わせ
SELECT 
    e.name AS 従業員名,
    p.name AS プロジェクト名
FROM employees e
CROSS JOIN projects p
ORDER BY e.name, p.name;

-- 実際の割り当てと比較
SELECT 
    e.name AS 従業員名,
    p.name AS プロジェクト名,
    CASE 
        WHEN ep.employee_id IS NOT NULL THEN '割当済'
        ELSE '未割当'
    END AS 状態
FROM employees e
CROSS JOIN projects p
LEFT JOIN employee_projects ep 
    ON e.id = ep.employee_id AND p.id = ep.project_id
ORDER BY e.name, p.name;
```

### SELF JOIN

同じテーブルを自己結合する方法です。

```sql
-- 管理職テーブルの作成
ALTER TABLE employees ADD COLUMN manager_id INTEGER;

-- 管理職の設定
UPDATE employees SET manager_id = 1 WHERE id IN (2, 3);
UPDATE employees SET manager_id = 3 WHERE id = 4;

-- 従業員とその管理者を表示
SELECT 
    e.name AS 従業員名,
    m.name AS 管理者名
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- 階層構造の表示
SELECT 
    e1.name AS レベル1,
    e2.name AS レベル2,
    e3.name AS レベル3
FROM employees e1
LEFT JOIN employees e2 ON e2.manager_id = e1.id
LEFT JOIN employees e3 ON e3.manager_id = e2.id
WHERE e1.manager_id IS NULL;
```

## 5.3 集計関数とGROUP BY

### 基本的な集計

```sql
-- 部署ごとの統計
SELECT 
    d.name AS 部署名,
    COUNT(e.id) AS 従業員数,
    AVG(e.salary) AS 平均給与,
    MAX(e.salary) AS 最高給与,
    MIN(e.salary) AS 最低給与,
    SUM(e.salary) AS 給与総額
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name;

-- プロジェクトごとの参加人数と予算
SELECT 
    p.name AS プロジェクト名,
    COUNT(DISTINCT ep.employee_id) AS 参加人数,
    p.budget AS 予算,
    p.budget / COUNT(DISTINCT ep.employee_id) AS 一人当たり予算
FROM projects p
LEFT JOIN employee_projects ep ON p.id = ep.project_id
GROUP BY p.id, p.name, p.budget;
```

### GROUP BYとHAVING

HAVINGは集計結果に対する条件指定に使用します。

```sql
-- 平均給与が450000円以上の部署
SELECT 
    d.name AS 部署名,
    AVG(e.salary) AS 平均給与
FROM departments d
INNER JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name
HAVING AVG(e.salary) >= 450000;

-- 2人以上が参加しているプロジェクト
SELECT 
    p.name AS プロジェクト名,
    COUNT(ep.employee_id) AS 参加人数
FROM projects p
INNER JOIN employee_projects ep ON p.id = ep.project_id
GROUP BY p.id, p.name
HAVING COUNT(ep.employee_id) >= 2
ORDER BY 参加人数 DESC;
```

### ウィンドウ関数

SQLiteはバージョン3.25.0からウィンドウ関数をサポートしています。

```sql
-- 部署内での給与ランキング
SELECT 
    name AS 従業員名,
    department_id AS 部署ID,
    salary AS 給与,
    RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS 部署内順位,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS 全体順位
FROM employees
WHERE department_id IS NOT NULL;

-- 累積と移動平均
SELECT 
    name AS 従業員名,
    hire_date AS 入社日,
    salary AS 給与,
    SUM(salary) OVER (ORDER BY hire_date) AS 累積給与,
    AVG(salary) OVER (ORDER BY hire_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS 移動平均
FROM employees
ORDER BY hire_date;

-- 前後の値との比較
SELECT 
    name AS 従業員名,
    salary AS 現在給与,
    LAG(salary, 1) OVER (ORDER BY hire_date) AS 前の人の給与,
    LEAD(salary, 1) OVER (ORDER BY hire_date) AS 次の人の給与,
    salary - LAG(salary, 1) OVER (ORDER BY hire_date) AS 給与差
FROM employees
ORDER BY hire_date;
```

## 5.4 サブクエリ

### SELECT句でのサブクエリ

```sql
-- 各従業員の給与と部署平均の比較
SELECT 
    e.name AS 従業員名,
    e.salary AS 給与,
    (SELECT AVG(salary) 
     FROM employees e2 
     WHERE e2.department_id = e.department_id) AS 部署平均,
    e.salary - (SELECT AVG(salary) 
                FROM employees e2 
                WHERE e2.department_id = e.department_id) AS 差額
FROM employees e
WHERE e.department_id IS NOT NULL;

-- 各部署の最高給与者
SELECT 
    d.name AS 部署名,
    (SELECT e.name 
     FROM employees e 
     WHERE e.department_id = d.id 
     ORDER BY e.salary DESC 
     LIMIT 1) AS 最高給与者,
    (SELECT MAX(e.salary) 
     FROM employees e 
     WHERE e.department_id = d.id) AS 最高給与
FROM departments d;
```

### FROM句でのサブクエリ

```sql
-- 部署別統計を使った分析
SELECT 
    dept_stats.部署名,
    dept_stats.平均給与,
    CASE 
        WHEN dept_stats.平均給与 > total_avg.avg_salary THEN '平均以上'
        ELSE '平均以下'
    END AS 評価
FROM (
    SELECT 
        d.name AS 部署名,
        AVG(e.salary) AS 平均給与
    FROM departments d
    INNER JOIN employees e ON d.id = e.department_id
    GROUP BY d.id, d.name
) AS dept_stats
CROSS JOIN (
    SELECT AVG(salary) AS avg_salary
    FROM employees
) AS total_avg;

-- プロジェクト参加状況の分析
SELECT 
    参加状況.参加プロジェクト数,
    COUNT(*) AS 該当従業員数
FROM (
    SELECT 
        e.id,
        e.name,
        COUNT(ep.project_id) AS 参加プロジェクト数
    FROM employees e
    LEFT JOIN employee_projects ep ON e.id = ep.employee_id
    GROUP BY e.id, e.name
) AS 参加状況
GROUP BY 参加状況.参加プロジェクト数
ORDER BY 参加状況.参加プロジェクト数;
```

### WHERE句でのサブクエリ

```sql
-- 平均給与以上の従業員
SELECT 
    name AS 従業員名,
    salary AS 給与
FROM employees
WHERE salary >= (SELECT AVG(salary) FROM employees)
ORDER BY salary DESC;

-- プロジェクトに参加している従業員
SELECT 
    name AS 従業員名,
    department_id AS 部署ID
FROM employees
WHERE id IN (
    SELECT DISTINCT employee_id 
    FROM employee_projects
);

-- 最も予算の多いプロジェクトに参加している従業員
SELECT 
    e.name AS 従業員名,
    p.name AS プロジェクト名
FROM employees e
INNER JOIN employee_projects ep ON e.id = ep.employee_id
INNER JOIN projects p ON ep.project_id = p.id
WHERE p.budget = (SELECT MAX(budget) FROM projects);
```

### 相関サブクエリ

```sql
-- 各部署で平均以上の給与を得ている従業員
SELECT 
    e1.name AS 従業員名,
    e1.salary AS 給与,
    d.name AS 部署名
FROM employees e1
INNER JOIN departments d ON e1.department_id = d.id
WHERE e1.salary >= (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e1.department_id
);

-- EXISTS/NOT EXISTSを使用
SELECT 
    e.name AS 従業員名
FROM employees e
WHERE EXISTS (
    SELECT 1
    FROM employee_projects ep
    INNER JOIN projects p ON ep.project_id = p.id
    WHERE ep.employee_id = e.id
    AND p.budget > 5000000
);

-- プロジェクトに参加していない従業員
SELECT 
    e.name AS 従業員名,
    e.department_id AS 部署ID
FROM employees e
WHERE NOT EXISTS (
    SELECT 1
    FROM employee_projects ep
    WHERE ep.employee_id = e.id
);
```

## 5.5 ビュー（VIEW）

### ビューの作成

```sql
-- 部署別統計ビュー
CREATE VIEW v_department_stats AS
SELECT 
    d.id AS department_id,
    d.name AS department_name,
    d.location,
    COUNT(e.id) AS employee_count,
    AVG(e.salary) AS avg_salary,
    MAX(e.salary) AS max_salary,
    MIN(e.salary) AS min_salary
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name, d.location;

-- 従業員詳細ビュー
CREATE VIEW v_employee_details AS
SELECT 
    e.id,
    e.name AS employee_name,
    e.salary,
    e.hire_date,
    d.name AS department_name,
    m.name AS manager_name,
    COUNT(DISTINCT ep.project_id) AS project_count
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
LEFT JOIN employees m ON e.manager_id = m.id
LEFT JOIN employee_projects ep ON e.id = ep.employee_id
GROUP BY e.id, e.name, e.salary, e.hire_date, d.name, m.name;

-- プロジェクト参加状況ビュー
CREATE VIEW v_project_participation AS
SELECT 
    p.id AS project_id,
    p.name AS project_name,
    p.budget,
    p.start_date,
    p.end_date,
    GROUP_CONCAT(e.name, ', ') AS participants,
    COUNT(ep.employee_id) AS participant_count
FROM projects p
LEFT JOIN employee_projects ep ON p.id = ep.project_id
LEFT JOIN employees e ON ep.employee_id = e.id
GROUP BY p.id, p.name, p.budget, p.start_date, p.end_date;
```

### ビューの使用

```sql
-- ビューからのSELECT
SELECT * FROM v_department_stats
WHERE avg_salary > 450000;

-- ビューを使ったJOIN
SELECT 
    ed.employee_name,
    ed.salary,
    ds.avg_salary AS dept_avg_salary,
    ed.salary - ds.avg_salary AS salary_diff
FROM v_employee_details ed
INNER JOIN v_department_stats ds 
    ON ed.department_name = ds.department_name
ORDER BY salary_diff DESC;

-- ビューの入れ子
CREATE VIEW v_high_budget_projects AS
SELECT * FROM v_project_participation
WHERE budget > 5000000;

SELECT * FROM v_high_budget_projects;
```

### ビューの管理

```sql
-- ビューの一覧表示
SELECT name FROM sqlite_master 
WHERE type = 'view';

-- ビューの定義確認
SELECT sql FROM sqlite_master 
WHERE type = 'view' AND name = 'v_department_stats';

-- ビューの削除
DROP VIEW IF EXISTS v_high_budget_projects;

-- ビューの再作成
DROP VIEW IF EXISTS v_department_stats;
CREATE VIEW v_department_stats AS
-- 新しい定義...
```

## 5.6 高度なテクニック

### WITH句（共通テーブル式：CTE）

```sql
-- 階層的なデータの処理
WITH RECURSIVE employee_hierarchy AS (
    -- アンカー：トップレベル（マネージャーなし）
    SELECT 
        id,
        name,
        manager_id,
        0 AS level,
        name AS path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- 再帰部分
    SELECT 
        e.id,
        e.name,
        e.manager_id,
        h.level + 1,
        h.path || ' > ' || e.name
    FROM employees e
    INNER JOIN employee_hierarchy h ON e.manager_id = h.id
)
SELECT * FROM employee_hierarchy
ORDER BY path;

-- 複数のCTEを使用
WITH 
dept_avg AS (
    SELECT 
        department_id,
        AVG(salary) AS avg_salary
    FROM employees
    WHERE department_id IS NOT NULL
    GROUP BY department_id
),
high_earners AS (
    SELECT 
        e.*,
        da.avg_salary AS dept_avg
    FROM employees e
    INNER JOIN dept_avg da ON e.department_id = da.department_id
    WHERE e.salary > da.avg_salary
)
SELECT 
    he.name,
    he.salary,
    he.dept_avg,
    d.name AS department_name
FROM high_earners he
INNER JOIN departments d ON he.department_id = d.id
ORDER BY he.salary DESC;
```

### CASE文の高度な使用

```sql
-- 複雑な条件分岐
SELECT 
    name AS 従業員名,
    salary AS 給与,
    CASE 
        WHEN salary >= 500000 THEN 'A'
        WHEN salary >= 450000 THEN 'B'
        WHEN salary >= 400000 THEN 'C'
        ELSE 'D'
    END AS 給与ランク,
    CASE 
        WHEN department_id = 1 AND salary > 450000 THEN '開発部エース'
        WHEN department_id = 2 AND salary > 450000 THEN '営業部エース'
        WHEN department_id IS NULL THEN '部署未配属'
        ELSE '一般社員'
    END AS 評価
FROM employees;

-- 動的な集計
SELECT 
    d.name AS 部署名,
    COUNT(CASE WHEN e.salary >= 500000 THEN 1 END) AS 高給与者数,
    COUNT(CASE WHEN e.salary < 500000 AND e.salary >= 450000 THEN 1 END) AS 中給与者数,
    COUNT(CASE WHEN e.salary < 450000 THEN 1 END) AS 低給与者数,
    COUNT(e.id) AS 総従業員数
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name;
```

### UNION、INTERSECT、EXCEPT

```sql
-- UNION: 結果を結合（重複除去）
SELECT name, '従業員' AS type FROM employees
UNION
SELECT name, '部署' AS type FROM departments
UNION
SELECT name, 'プロジェクト' AS type FROM projects
ORDER BY type, name;

-- UNION ALL: 重複を含めて結合
SELECT department_id, '給与500000以上' AS category 
FROM employees WHERE salary >= 500000
UNION ALL
SELECT department_id, '給与450000以上' AS category 
FROM employees WHERE salary >= 450000;

-- INTERSECT: 共通部分
SELECT employee_id FROM employee_projects WHERE project_id = 1
INTERSECT
SELECT employee_id FROM employee_projects WHERE project_id = 3;

-- EXCEPT: 差集合
SELECT id FROM employees
EXCEPT
SELECT employee_id FROM employee_projects;
```

## 5.7 パフォーマンスを考慮したクエリ

### インデックスを活用したクエリ

```sql
-- インデックスの作成
CREATE INDEX idx_employees_department ON employees(department_id);
CREATE INDEX idx_employees_salary ON employees(salary);
CREATE INDEX idx_employee_projects ON employee_projects(employee_id, project_id);

-- 複合インデックスの活用
CREATE INDEX idx_employees_dept_salary ON employees(department_id, salary);

-- カバリングインデックス
CREATE INDEX idx_employees_covering ON employees(department_id, name, salary);
```

### クエリの最適化

```sql
-- 非効率なクエリ
SELECT * FROM employees 
WHERE department_id IN (
    SELECT id FROM departments WHERE location = '東京'
);

-- 最適化されたクエリ
SELECT e.* 
FROM employees e
INNER JOIN departments d ON e.department_id = d.id
WHERE d.location = '東京';

-- EXPLAINを使った実行計画の確認
EXPLAIN QUERY PLAN
SELECT e.name, d.name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id
WHERE d.location = '東京';
```

## 5.8 実践的な例

### 月次レポートの作成

```sql
-- 部署別月次サマリー
CREATE VIEW v_monthly_department_summary AS
WITH project_costs AS (
    SELECT 
        e.department_id,
        SUM(p.budget * 1.0 / 
            (julianday(p.end_date) - julianday(p.start_date))) AS daily_cost
    FROM employee_projects ep
    INNER JOIN employees e ON ep.employee_id = e.id
    INNER JOIN projects p ON ep.project_id = p.id
    GROUP BY e.department_id
)
SELECT 
    d.name AS 部署名,
    ds.employee_count AS 従業員数,
    ds.avg_salary AS 平均給与,
    ds.avg_salary * ds.employee_count AS 月間人件費,
    COALESCE(pc.daily_cost * 30, 0) AS プロジェクト予算（月換算）,
    (ds.avg_salary * ds.employee_count + COALESCE(pc.daily_cost * 30, 0)) AS 総コスト
FROM v_department_stats ds
INNER JOIN departments d ON ds.department_id = d.id
LEFT JOIN project_costs pc ON ds.department_id = pc.department_id;
```

### ダッシュボード用データ

```sql
-- 総合ダッシュボード
WITH summary AS (
    SELECT 
        (SELECT COUNT(*) FROM employees) AS total_employees,
        (SELECT COUNT(*) FROM departments) AS total_departments,
        (SELECT COUNT(*) FROM projects) AS total_projects,
        (SELECT AVG(salary) FROM employees) AS avg_salary,
        (SELECT SUM(budget) FROM projects) AS total_budget
)
SELECT 
    '従業員総数' AS 項目,
    total_employees AS 値
FROM summary
UNION ALL
SELECT '部署数', total_departments FROM summary
UNION ALL
SELECT '実施中プロジェクト数', total_projects FROM summary
UNION ALL
SELECT '平均給与', ROUND(avg_salary) FROM summary
UNION ALL
SELECT '総プロジェクト予算', total_budget FROM summary;
```

## 5.9 まとめ

この章では、SQLiteにおける高度なクエリ技術を学びました。JOIN操作、集計関数、サブクエリ、ビューなどを組み合わせることで、複雑なデータ分析や報告書の作成が可能になります。

### この章で学んだこと

- 各種JOIN（INNER、LEFT、CROSS、SELF）の使い方
- GROUP BYとHAVINGによる集計
- ウィンドウ関数による高度な分析
- サブクエリとCTEの活用
- ビューによるクエリの再利用
- パフォーマンスを考慮したクエリ設計

### 確認問題

1. INNER JOINとLEFT JOINの違いを説明し、それぞれが適している場面を述べてください
2. 各部署の最高給与者と最低給与者を表示するクエリを作成してください
3. 2つ以上のプロジェクトに参加している従業員のリストを作成してください
4. WITH句を使用して、部署階層を表示するクエリを作成してください

---

**[← 第4章: データ型と制約へ戻る](./chapter04-datatypes-constraints.md)** | **[→ 第6章: インデックスとパフォーマンスへ進む](./chapter06-indexes-performance.md)**