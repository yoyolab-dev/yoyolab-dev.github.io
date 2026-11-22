---
title: 'CSV 里有逗号和换行时，cut 就不够用了'
date: 2026-11-22 12:18:24
tags: [Python, CSV, 命令行, 数据处理]
categories: [技术教程]
---

今天整理一张只有两条订单的 CSV，我被 `cut` 教育了一次。表格看着很普通：姓名、商品、价格、备注；其中一条商品写着 `tea, cake`，备注里还有引号和换行。`cut` 只认逗号，不认引号，于是价格列被整个挤歪了。

先把这个问题缩成一个能复现的样本。下面这条命令会在当前目录生成 `orders.csv`，故意把逗号、双引号和换行都塞进字段里：

```bash
python3 -c 'import csv; rows=[["name","item","price","note"],["Ada","tea, cake","18.5","said \"hello\"\nthen left"],["Lin","tea","12","ok"]]; f=open("orders.csv","w",newline="",encoding="utf-8"); csv.writer(f).writerows(rows); f.close()'
```

命令正常结束时没有输出。用 `wc -l` 看物理行数：

```console
$ wc -l orders.csv
4 orders.csv
```

这里是 4 行，但 CSV 实际只有 3 条记录：1 行表头加 2 行数据。第二条数据的备注中藏着一个换行，所以编辑器把它显示成了两行。别把物理行数当记录数，后续对账会从这里开始错。

我先把常见的切列命令拿出来试。它想取第三列 `price`：

```console
$ cut -d, -f3 orders.csv
price
 cake"
then left"
12
```

本该出现的 `18.5` 不见了，反而混进了 ` cake"` 和 `then left"`。原因很直接：`cut` 会把每一行都独立按逗号切开，无法知道 `tea, cake` 外面的双引号表示“这是一个字段”。失败现象就是列错位、记录数对不上、总数不对；这时继续修改 `-F,` 没用，分隔符本身没有错，错的是解析器不懂 CSV 的引号规则。

修法是让 Python 自带的 `csv` 模块先找出列边界：

```console
$ python3 -c 'import csv,sys; rows=list(csv.reader(sys.stdin)); print("records =",len(rows)); [print(i,len(r),r) for i,r in enumerate(rows,1)]' < orders.csv
records = 3
1 4 ['name', 'item', 'price', 'note']
2 4 ['Ada', 'tea, cake', '18.5', 'said "hello"\nthen left']
3 4 ['Lin', 'tea', '12', 'ok']
```

判定标准很清楚：`records` 是 3，每一行的字段数都是 4；逗号、双引号和 `\n` 都回到了原字段里，没有再切碎。

真正做统计时，我更常用 `DictReader`，它会把表头变成列名，不用再靠数字猜位置：

```console
$ python3 -c 'import csv,sys; rows=list(csv.DictReader(sys.stdin)); print("data rows =",len(rows)); print("total =",sum(float(r["price"]) for r in rows)); print("notes =",[r["note"] for r in rows])' < orders.csv
data rows = 2
total = 30.5
notes = ['said "hello"\nthen left', 'ok']
```

这里 `data rows = 2`，正好是两条数据；金额合计 `30.5`，等于 `18.5 + 12`；备注列表也完整保留了换行。`csv.reader` 把表头算进 3 条记录，`DictReader` 只统计数据行，所以两个数字相差 1，不是解析打架。

以前我觉得 `cut` 简单可靠，直到它把一条记录劈成两半。现在我遇到带引号、逗号或换行的 CSV，会先让 `csv` 模块告诉我列边界，再把整理好的结果交给别的工具。那几行 `1 4`、`2 4`、`3 4` 对齐得整整齐齐，比表格里看着正常的列更让人安心。
