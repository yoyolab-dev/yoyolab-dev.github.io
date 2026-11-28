---
title: '两张表按名字牵到了一起：join 的用法和一次排序坑'
date: 2026-11-28 19:18:55
tags:
  - Linux
  - 命令行
  - 文本处理
categories:
  - 技术
---

今天我手里有两张很小的表：一张写着机器名和它负责的端口，另一张写着机器名和它所在的城市。两张表都按机器名排好了，可我还是想按名字把它们接起来，而不是凭屏幕上的位置硬猜。Linux 自带的 `join` 正好干这件事。

先造一份能复现的样例。以下命令放进空目录执行：

```bash
mkdir join-demo
cd join-demo
printf 'host\tport\nalpha\t8080\nbeta\t9090\ngamma\t3000\n' > ports.tsv
printf 'host\tcity\nbeta\tShanghai\ngamma\tSuzhou\nalpha\tHangzhou\n' > cities.tsv
```

先别急着接。我故意把 `cities.tsv` 留成 `beta、gamma、alpha` 的顺序，然后运行：

```bash
LC_ALL=C join -t $'\t' -1 1 -2 1 ports.tsv cities.tsv
```

实际输出只有一行：

```text
beta	9090	Shanghai
```

为什么只剩 `beta`？因为 `join` 不会把两张表擅自整理好，它要求连接字段已经按相同的字节序排序。`ports.tsv` 是 `alpha、beta、gamma`，另一张却从 `beta` 开始，比较器一遇到乱序就直接停止；后面的 `gamma` 和 `alpha` 再也碰不到一起。

失败时它不会写一行“排序错误”，只是安静地少给结果。我拿输出行数核对过：

```bash
LC_ALL=C join -t $'\t' -1 1 -2 1 ports.tsv cities.tsv | wc -l
```

结果是 `1`，而不是应有的 `3`。这种“命令成功退出，结果却少一半”的现象，比直接报错更值得警惕。

修法是在连接前分别排序。我用临时文件，不覆盖原表：

```bash
LC_ALL=C sort -t $'\t' -k1,1 ports.tsv > ports.sorted
LC_ALL=C sort -t $'\t' -k1,1 cities.tsv > cities.sorted
LC_ALL=C join -t $'\t' -1 1 -2 1 ports.sorted cities.sorted
```

这次三行都回来了：

```text
alpha	8080	Hangzhou
beta	9090	Shanghai
gamma	3000	Suzhou
```

判定标准可以写成一句：连接后的行数应等于两张表中共有键的数量。这里用 `cut` 分别抽出第一列，再数交集，正好是 3：

```bash
printf 'matched=%s\n' \
  "$(comm -12 <(cut -f1 ports.sorted) <(cut -f1 cities.sorted) | wc -l)"
```

输出为 `matched=3`。另外，两张表的键必须确实用同一个字段时，`-1 1 -2 1` 可以省掉；我保留它，是为了让“按第一列连接”这件事写在命令里，不靠猜。

如果两张表都已经按连接键排序，我还可以省掉中间文件：

```bash
LC_ALL=C join -t $'\t' -1 1 -2 1 \
  <(LC_ALL=C sort -t $'\t' -k1,1 ports.tsv) \
  <(LC_ALL=C sort -t $'\t' -k1,1 cities.tsv)
```

输出仍是上面那三行。这里要注意，排序区域和连接区域是两个独立的步骤；我第一次把只排过一边时就以为能省掉另一边，结果又只得到一行。

`join` 默认还会把重复键做乘法。我把 `alpha` 在端口表里故意多放一行，再连接一次，两边的 `alpha` 会各自配对，得到两行 `alpha`。所以它不是普通的字典查询：一个键出现几次，输出就可能膨胀几次。

两张表一共只有三台机器，最后并排看到 `alpha、beta、gamma` 时，我还是觉得比在一列列数字母靠谱。安静少给答案的命令最麻烦，它看起来像成功，只有拿预期行数去数，才抓得住这次少掉的 `gamma` 和 `alpha`。
