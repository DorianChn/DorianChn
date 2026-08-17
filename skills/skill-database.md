# Python SQLite 数据库操作

> 用Python内置的sqlite3模块，零安装搞定数据存储和查询

## 这个skill能做什么

用Python操作SQLite数据库：创建表、增删改查数据、批量导入导出，**不需要安装任何数据库软件**。

## 使用场景

- 保存**爬虫数据**（网页内容、用户信息）
- 记录**日志数据**（运行日志、访问记录）
- 搭建**本地应用**的数据存储（记账本、图书管理）
- 数据分析前的**数据清洗和整理**
- 小网站/小工具的**后端数据库**

## 前置要求

```bash
# 什么都不用装！Python自带sqlite3模块
# 验证一下：
python -c "import sqlite3; print('✅ 已就绪')"
```

## 快速开始

```bash
# 1. 保存下面代码为 demo.py
# 2. 运行
python demo.py
# 3. 会生成一个 student.db 文件，这就是数据库
```

## 完整代码

### 1️⃣ 基础操作：创建表 + 增删改查

```python
"""
功能：SQLite数据库完整操作示例
运行：python demo.py
效果：创建一个学生信息数据库，演示CRUD操作
"""
import sqlite3
import os

# ===== 1. 连接数据库（没有则自动创建） =====
DB_FILE = "student.db"
conn = sqlite3.connect(DB_FILE)
cursor = conn.cursor()
print(f"✅ 连接数据库: {DB_FILE}")

# ===== 2. 创建表 =====
cursor.execute("""
CREATE TABLE IF NOT EXISTS students (
    id INTEGER PRIMARY KEY AUTOINCREMENT,  -- 自动编号
    name TEXT NOT NULL,                      -- 姓名（必填）
    age INTEGER,                            -- 年龄
    score REAL DEFAULT 0,                   -- 成绩（默认0分）
    city TEXT                               -- 城市
)
""")
conn.commit()
print("✅ 创建表: students")

# ===== 3. 插入数据 =====
print("\n📝 插入数据...")

# 插入单条
cursor.execute("INSERT INTO students (name, age, score, city) VALUES (?, ?, ?, ?)",
               ("张三", 20, 88.5, "北京"))
print("  + 张三")

# 插入多条
students_data = [
    ("李四", 22, 92.0, "上海"),
    ("王五", 19, 76.5, "广州"),
    ("赵六", 21, 95.0, "深圳"),
    ("钱七", 20, 83.0, "杭州"),
]
cursor.executemany("INSERT INTO students (name, age, score, city) VALUES (?, ?, ?, ?)",
                   students_data)
print(f"  + 批量插入 {len(students_data)} 条")
conn.commit()

# ===== 4. 查询数据 =====
print("\n🔍 查询所有学生:")
cursor.execute("SELECT * FROM students")
for row in cursor.fetchall():
    print(f"  {row[0]}. {row[1]} | {row[2]}岁 | {row[3]}分 | {row[4]}")

# 条件查询
print("\n🔍 查询成绩>80分的学生:")
cursor.execute("SELECT name, score FROM students WHERE score > ? ORDER BY score DESC", (80,))
for row in cursor.fetchall():
    print(f"  {row[0]}: {row[1]}分")

# ===== 5. 更新数据 =====
print("\n✏️ 更新数据...")
cursor.execute("UPDATE students SET score = ? WHERE name = ?", (90.0, "王五"))
print("  王五成绩更新为90分")
conn.commit()

# ===== 6. 删除数据 =====
# print("\n🗑️ 删除数据...")
# cursor.execute("DELETE FROM students WHERE name = ?", ("钱七",))
# print("  删除钱七")
# conn.commit()

# ===== 7. 统计查询 =====
print("\n📊 统计信息:")
cursor.execute("""
SELECT 
    COUNT(*) as 总人数,
    ROUND(AVG(score), 1) as 平均分,
    MAX(score) as 最高分,
    MIN(score) as 最低分
FROM students
""")
row = cursor.fetchone()
print(f"  总人数: {row[0]}")
print(f"  平均分: {row[1]}")
print(f"  最高分: {row[2]}")
print(f"  最低分: {row[3]}")

# ===== 8. 关闭连接 =====
cursor.close()
conn.close()
print(f"\n✅ 操作完成！数据库文件: {DB_FILE}")
print(f"   文件大小: {os.path.getsize(DB_FILE) / 1024:.1f} KB")
```

### 2️⃣ 实用场景：数据导入导出

```python
"""
功能：CSV文件与SQLite数据库互转
场景：把Excel导出的CSV数据导入数据库，或把数据库导出为CSV
"""
import sqlite3
import csv
import os

def csv_to_sqlite(csv_file, db_file, table_name):
    """
    把CSV文件导入到SQLite数据库
    
    参数:
        csv_file: CSV文件路径
        db_file: 数据库文件路径
        table_name: 表名
    """
    conn = sqlite3.connect(db_file)
    cursor = conn.cursor()
    
    # 读取CSV
    with open(csv_file, "r", encoding="utf-8") as f:
        reader = csv.reader(f)
        headers = next(reader)  # 第一行是列名
        rows = list(reader)     # 剩余是数据
    
    # 动态创建表（列名 = CSV表头，类型 = TEXT）
    columns = ", ".join([f'"{h}" TEXT' for h in headers])
    cursor.execute(f"DROP TABLE IF EXISTS {table_name}")
    cursor.execute(f"CREATE TABLE {table_name} ({columns})")
    
    # 插入数据
    placeholders = ", ".join(["?" for _ in headers])
    cursor.executemany(
        f"INSERT INTO {table_name} VALUES ({placeholders})", rows
    )
    conn.commit()
    cursor.close()
    conn.close()
    print(f"✅ CSV → SQLite: {len(rows)} 行数据导入到 {table_name}")

def sqlite_to_csv(db_file, table_name, csv_file):
    """把SQLite表导出为CSV文件"""
    conn = sqlite3.connect(db_file)
    cursor = conn.cursor()
    
    cursor.execute(f"SELECT * FROM {table_name}")
    rows = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    
    with open(csv_file, "w", encoding="utf-8", newline="") as f:
        writer = csv.writer(f)
        writer.writerow(columns)  # 写入表头
        writer.writerows(rows)    # 写入数据
    
    cursor.close()
    conn.close()
    print(f"✅ SQLite → CSV: {len(rows)} 行数据导出到 {csv_file}")

# ----- 使用示例 -----
if __name__ == "__main__":
    # 生成示例CSV
    with open("demo_data.csv", "w", encoding="utf-8", newline="") as f:
        w = csv.writer(f)
        w.writerow(["姓名", "年龄", "城市", "工资"])
        w.writerow(["张三", 28, "北京", 15000])
        w.writerow(["李四", 32, "上海", 18000])
        w.writerow(["王五", 25, "广州", 12000])
    
    # 导入到数据库
    csv_to_sqlite("demo_data.csv", "company.db", "employees")
    
    # 导出回CSV
    sqlite_to_csv("company.db", "employees", "exported_data.csv")
    
    # 清理临时文件
    # os.remove("demo_data.csv")
    # os.remove("exported_data.csv")
    # os.remove("company.db")
```

### 3️⃣ 实用场景：分页查询（大数据量）

```python
"""
功能：大数据量分页查询，避免一次性加载太多数据
"""
import sqlite3

def paginate_query(db_file, table_name, page_size=10):
    """分页查询演示"""
    conn = sqlite3.connect(db_file)
    cursor = conn.cursor()
    
    # 先查总条数
    cursor.execute(f"SELECT COUNT(*) FROM {table_name}")
    total = cursor.fetchone()[0]
    total_pages = (total + page_size - 1) // page_size
    
    print(f"📊 共 {total} 条数据，每页 {page_size} 条，共 {total_pages} 页\n")
    
    page = 1
    while True:
        # 计算偏移量
        offset = (page - 1) * page_size
        
        cursor.execute(
            f"SELECT * FROM {table_name} LIMIT ? OFFSET ?",
            (page_size, offset)
        )
        rows = cursor.fetchall()
        
        print(f"--- 第 {page}/{total_pages} 页 ---")
        for row in rows:
            print(f"  {row}")
        
        print(f"  [第 {page} 页，显示 {len(rows)} 条]\n")
        
        # 下一页
        if page < total_pages:
            cmd = input("按 Enter 看下一页，输入 q 退出: ")
            if cmd.lower() == "q":
                break
            page += 1
        else:
            print("✅ 已是最后一页")
            break
    
    cursor.close()
    conn.close()

# 使用：先生成测试数据，再分页查看
if __name__ == "__main__":
    # 生成100条测试数据
    conn = sqlite3.connect("test.db")
    c = conn.cursor()
    c.execute("CREATE TABLE IF NOT EXISTS logs (id INTEGER, msg TEXT, time TEXT)")
    for i in range(1, 101):
        c.execute("INSERT INTO logs VALUES (?, ?, ?)",
                  (i, f"日志第{i}条", f"2024-01-{i%30+1:02d}"))
    conn.commit()
    conn.close()
    print("✅ 已生成100条测试数据")
    
    # 分页查询
    paginate_query("test.db", "logs", page_size=10)
```

## 常见问题

**Q: 运行报错 `no such table`**
A: 表还没创建。先执行 `CREATE TABLE` 再查询。或者检查表名是否拼写正确（区分大小写）。

**Q: 中文显示乱码**
A: 连接数据库时加参数：`sqlite3.connect(DB_FILE)` 默认就是UTF-8。如果CSV文件乱码，把 `encoding="utf-8"` 改成 `encoding="gbk"`。

**Q: 数据库文件在哪？**
A: 就在你运行Python脚本的目录下，生成一个 `.db` 文件。可以直接用 [DB Browser for SQLite](https://sqlitebrowser.org/) 打开查看。

**Q: 如何删除整个数据库？**
A: 直接删除 `.db` 文件就行。SQLite就是一个文件，删了就没了。

**Q: 如何备份？**
A: 复制 `.db` 文件就是备份。或者用 `sqlite3 db_file.db ".dump" > backup.sql` 导出SQL。

**Q: 多用户同时访问怎么办？**
A: SQLite支持并发读，但不适合高并发写。如果多用户同时写，建议用MySQL/PostgreSQL。

## 进阶用法

- **SQLAlchemy ORM**：用面向对象的方式操作数据库，不用写SQL
- **数据库连接池**：高并发场景下复用连接，提升性能
- **全文搜索**：SQLite内置FTS5，支持中文全文搜索
- **加密数据库**：使用 `sqlcipher` 给数据库加密
- **内存数据库**：`:memory:` 连接模式，数据只存在内存中，程序退出即消失

## 参考资源

- [SQLite 官方教程](https://www.sqlitetutorial.net/) - 最详细的SQLite教程
- [SQLite 官方文档 - Python](https://docs.python.org/3/library/sqlite3.html) - Python sqlite3模块文档
- [DB Browser for SQLite](https://sqlitebrowser.org/) - 免费的数据库可视化工具
- [SQL 在线练习](https://www.w3schools.com/sql/) - 在线写SQL，即时看结果
- [SQLite 语法速查](https://www.runoob.com/sqlite/sqlite-tutorial.html) - 中文版SQLite教程

---

> 💡 **小贴士**：SQLite是世界上最流行的数据库，你的手机里每个App都在用。学会SQLite = 学会所有SQL数据库的基础！