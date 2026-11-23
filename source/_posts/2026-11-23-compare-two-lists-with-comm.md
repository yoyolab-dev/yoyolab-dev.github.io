---
title: '两份清单差在哪：sort、comm 和三种差集'
date: 2026-11-23 10:18:24
tags:
  - Linux
  - 命令行
  - 文本处理
categories:
  - 技术
---

今天我有一件很笨的事要做：把两份软件清单对一遍，一份是机器上已经有的，一份是我希望它有的。以前我会用眼睛一行行找，盯着屏幕久了反而容易漏。现在我更愿意把这件事交给 `comm`，它能老老实实告诉我三件事：两边都有、左边独有、右边独有。

先造两份能复现的样例。以下命令放进一个空目录执行：

```bash
mkdir comm-demo
cd comm-demo
printf 'curl\nnode\npython\nnginx\nredis\n' > left.txt
printf 'curl\npython\nnginx\ndocker\nredis\n' > right.txt
```

`left.txt` 和 `right.txt` 各有 5 行，肉眼看只有 `node` 和 `docker` 的位置不同。先别直接运行 `comm`，它要求输入已经排好序；没排序时，它可能给出错误的结果，也可能只打一句警告就继续跑。我先把两份清单各自排序并去重：

```bash
LC_ALL=C sort -u left.txt > left.sorted
LC_ALL=C sort -u right.txt > right.sorted
wc -l left.sorted right.sorted
```

实际输出是：

```text
5 left.sorted
5 right.sorted
```

现在 `comm` 才真正派上用场。三个数字表示要隐藏第几列：

```bash
comm -23 left.sorted right.sorted
comm -13 left.sorted right.sorted
comm -12 left.sorted right.sorted
```

依次得到：

```text
node
docker
curl
nginx
python
redis
```

第一条只留下左边独有的 `node`，第二条只留下右边独有的 `docker`，第三条是两边都有的 4 行。判定标准也很简单：共有项应为 `4`，左右独有项各为 `1`，加起来正好是每边的 5 行。

再确认数量：

```bash
printf 'common=%s left_only=%s right_only=%s\n' \
  "$(comm -12 left.sorted right.sorted | wc -l)" \
  "$(comm -23 left.sorted right.sorted | wc -l)" \
  "$(comm -13 left.sorted right.sorted | wc -l)"
```

输出是：

```text
common=4 left_only=1 right_only=1
```

如果嫌中间文件碍事，在 Bash 里也能直接用临时输入：

```bash
comm -23 <(LC_ALL=C sort -u left.txt) <(LC_ALL=C sort -u right.txt)
```

这里千万别忘了排序。我故意用没排过的原文件试过，`comm` 报出了 `file 1 is not in sorted order` 和 `file 2 is not in sorted order`，列出的内容也已经不能当作可靠差集。这就是它最值得注意的失败现象：不是一定停下，而是可能一边警告一边继续给错结果。

两份清单一共才 10 行，我以前却得来回扫好几遍。现在看见 `node` 和 `docker` 各自孤零零地站在结果里，倒比盯屏幕更放心。
