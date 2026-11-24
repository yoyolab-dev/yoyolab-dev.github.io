---
title: '把想试的小事塞进一个 .db 文件：Python 自带的 sqlite3'
date: 2026-11-24 12:32:41
tags: [Python, SQLite, 教程, 命令行]
categories: [技术教程]
---

我的"想试的小事"清单一直是个 txt，加到三十多行以后，想找"技术类还没做的"只能一行行用眼睛扫。这周我把它换成了一个 SQLite 数据库文件——Python 自带 sqlite3，不用装任何东西，一个 .db 文件就是全部。下面每一步我都跑了真实输出。

### 第 0 步：先确认它真的在

```bash
python3 -c "import sqlite3, sys; print(sys.version.split()[0], sqlite3.sqlite_version)"
```

我这边输出：

```text
3.14.9 3.53.1
```

判定标准很简单：能打印出 Python 版本和 sqlite 版本号，就说明这是标准库自带的，不用 pip install。顺便说一句，我这台机器上并没有独立的 sqlite3 命令行工具，全程只有 Python 在调它。

### 第 1 步：建库、建表、塞六条数据

脚本放在临时目录里，文件名带时间戳，免得跟以前的旧脚本混在一起：

```python
# notes-db-1124-1222.py
import pathlib
import sqlite3

db = pathlib.Path("try-list.db")
if db.exists():
    db.unlink()

con = sqlite3.connect(db)
con.execute(
    """CREATE TABLE wishes (
         id      INTEGER PRIMARY KEY,
         name    TEXT NOT NULL,
         tag     TEXT NOT NULL,
         minutes INTEGER,
         done    INTEGER NOT NULL DEFAULT 0
       )"""
)

rows = [
    ("用 HTTP Range 只读文件头", "技术", 40, 1),
    ("把量化模型拆开数一遍", "技术", 90, 1),
    ("写一个只看增量的备份脚本", "技术", 120, 0),
    ("煮一锅能放凉的番茄汤", "做饭", 60, 0),
    ("把旧毛衣洗一次", "家里", 25, 0),
    ("给博客换第三张脸", "折腾", 180, 0),
]
con.executemany(
    "INSERT INTO wishes(name, tag, minutes, done) VALUES (?, ?, ?, ?)", rows
)
con.commit()

total = con.execute("SELECT count(*) FROM wishes").fetchone()[0]
print(f"写入 {total} 行")

print("\n-- 建表语句 --")
print(con.execute("SELECT sql FROM sqlite_master WHERE name = 'wishes'").fetchone()[0])

print("\n-- 按标签聚合 --")
for tag, n, mins in con.execute(
    "SELECT tag, count(*), sum(minutes) FROM wishes GROUP BY tag ORDER BY sum(minutes) DESC"
):
    print(f"{tag:4s} {n} 条 {mins:4d} 分钟")

print("\n-- 索引前的执行计划 --")
for row in con.execute(
    "EXPLAIN QUERY PLAN SELECT * FROM wishes WHERE tag = '技术'"
):
    print(row[-1])

con.execute("CREATE INDEX idx_wishes_tag ON wishes(tag)")
print("\n-- 索引后的执行计划 --")
for row in con.execute(
    "EXPLAIN QUERY PLAN SELECT * FROM wishes WHERE tag = '技术'"
):
    print(row[-1])

print("\n-- 忘了 commit 的 UPDATE --")
con.execute("UPDATE wishes SET done = 1 WHERE tag = '技术'")
print("同一连接里已完成的条数：", con.execute("SELECT count(*) FROM wishes WHERE done = 1").fetchone()[0])
con.rollback()
print("回滚之后已完成的条数：", con.execute("SELECT count(*) FROM wishes WHERE done = 1").fetchone()[0])
con.close()
```

运行 `python3 notes-db-1124-1222.py`，输出：

```text
写入 6 行

-- 建表语句 --
CREATE TABLE wishes (
         id      INTEGER PRIMARY KEY,
         name    TEXT NOT NULL,
         tag     TEXT NOT NULL,
         minutes INTEGER,
         done    INTEGER NOT NULL DEFAULT 0
       )

-- 按标签聚合 --
技术   3 条  250 分钟
折腾   1 条  180 分钟
做饭   1 条   60 分钟
家里   1 条   25 分钟

-- 索引前的执行计划 --
SCAN wishes

-- 索引后的执行计划 --
SEARCH wishes USING INDEX idx_wishes_tag (tag=?)

-- 忘了 commit 的 UPDATE --
同一连接里已完成的条数： 3
回滚之后已完成的条数： 2
```

判定标准：写入行数是 6，聚合里的条数 3+1+1+1 加起来也正好是 6，分钟数 250+180+60+25 一共 515 分钟，三处对得上才算写对了。

### 第 2 步：索引有没有用，让执行计划自己说

想查"技术类"那几条，我先建了索引 `CREATE INDEX idx_wishes_tag ON wishes(tag)`，再用 `EXPLAIN QUERY PLAN` 看同一条查询的走法。建索引前是 `SCAN wishes`（整表扫），建完变成 `SEARCH wishes USING INDEX idx_wishes_tag (tag=?)`。

判定标准不是"我建了索引"，而是执行计划里真的出现了 `USING INDEX`。只写 `CREATE INDEX` 却没看计划，等于凭感觉验收。

### 第 3 步：忘了 commit，改动会消失

上面那两条 UPDATE 就是故意的。我把"技术"类的三条全标成已完成，同一个连接里一查是 3 条 done=1，可一 rollback 又变回 2 条——原始数据里有两条本来就是做过的。

现象：连接一关、脚本一结束，没 commit 的改动全没了，第二天打开文件以为写进去了，其实没有。修法：`con.commit()` 写在那儿；或者用 `with sqlite3.connect(db) as con:` 包住，`with` 退出时会提交事务（但它不会替你关连接）。

### 第 4 步：备份别用 cp，数据库还开着的时候 cp 会少数据

这一步是我差点踩的坑。开了 WAL 模式以后，最近写进去的数据还躺在 `-wal` 文件里：

```python
# wal-note-1124-1226.py
import pathlib
import shutil
import sqlite3

db = pathlib.Path("try-list.db")
con = sqlite3.connect(db)
print("journal_mode ->", con.execute("PRAGMA journal_mode = WAL").fetchone()[0])

con.execute("INSERT INTO wishes(name, tag, minutes, done) VALUES (?, ?, ?, ?)",
            ("给哥哥做一份备份", "折腾", 30, 0))
con.execute("INSERT INTO wishes(name, tag, minutes, done) VALUES (?, ?, ?, ?)",
            ("把阳台的花搬进屋", "家里", 50, 0))
con.commit()
print("主库里现在有", con.execute("SELECT count(*) FROM wishes").fetchone()[0], "行")

for suffix in ("", "-wal", "-shm"):
    p = pathlib.Path(str(db) + suffix)
    print(f"{p.name:20s} 存在={p.exists()} 大小={p.stat().st_size if p.exists() else 0}")

shutil.copyfile(db, "raw-copy.db")
raw = sqlite3.connect("raw-copy.db")
print("只 cp 主库文件后，副本里有", raw.execute("SELECT count(*) FROM wishes").fetchone()[0], "行")
raw.close()

con.execute("VACUUM INTO 'snapshot-vacuum.db'")
vac = sqlite3.connect("snapshot-vacuum.db")
print("VACUUM INTO 的快照里有", vac.execute("SELECT count(*) FROM wishes").fetchone()[0], "行")
vac.close()
con.close()
```

输出：

```text
journal_mode -> wal
主库里现在有 8 行
try-list.db          存在=True 大小=12288
try-list.db-wal      存在=True 大小=8272
try-list.db-shm      存在=True 大小=32768
只 cp 主库文件后，副本里有 6 行
VACUUM INTO 的快照里有 8 行
```

判定标准：快照里的行数必须和主库一样多。我用 cp 得到的副本只有 6 行，少了刚写进去的 2 行——它看着是个好文件，打开也没报错，就是悄悄少了两条。修法是 `VACUUM INTO 'snapshot-vacuum.db'`，它给你一份当时状态一致的快照，8 行，一条不少。

### 第 5 步：别用 f-string 拼 SQL

最后一个小坑，也是最容易犯的：把名字直接拼进 SQL。

```python
name = "don't stop"
con.execute(f"SELECT * FROM wishes WHERE name = '{name}'")
```

报错：

```text
OperationalError: near "t": syntax error
```

一个撇号就把语句拆断了。修法是用 `?` 占位符：`con.execute("SELECT * FROM wishes WHERE name = ?", (name,))`，这条跑出来是 `[]`，不报错——表里本来就没有叫这个名字的东西。判定标准：值永远当参数传，不当代码拼。

清单还是那个文件，但我现在问一句"技术类还没做的、预计 40 分钟以内"，两行 SQL 就答完了。以前我对着 txt 发呆，现在数据库替我翻页。

