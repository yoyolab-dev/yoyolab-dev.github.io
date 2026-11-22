---
title: 'find 里的“昨天”，为什么差一点'
date: 2026-11-22 19:25:30
tags:
  - Linux
  - find
  - 命令行
categories:
  - 技术
---

我今天整理旧文件时，顺手写了句 `find . -mtime 1`，心里想的是“找昨天改过的东西”。结果那个已经 25 小时没动过的文件冒出来了，昨天刚改过的一个反而没影。

原来 `find` 看的不是日历上的昨天，而是先算出文件隔了多少个完整的 24 小时，再把零头扔掉。这个坑不咬人一下还真看不出来，我干脆搭了三个样本复现了一遍。

### 先造三个看得见年龄的文件

下面这段在 GNU/Linux 上跑，`find` 版本是 4.10.0：

```bash
mkdir -p ~/find-mtime-demo
cd ~/find-mtime-demo
now=$(date +%s)
touch -d "@$((now-30*60))" 30-minutes-old
touch -d "@$((now-25*3600))" 25-hours-old
touch -d "@$((now-49*3600))" 49-hours-old

stat -c '%n  %y' 30-minutes-old 25-hours-old 49-hours-old
```

我这里实际看到的是：

```text
30-minutes-old  2026-11-22 18:53:44.000000000 +0800
25-hours-old    2026-11-21 18:23:44.000000000 +0800
49-hours-old    2026-11-20 18:23:44.000000000 +0800
```

接下来分别查询：

```bash
find . -maxdepth 1 -type f -mtime 0 -printf '%f\n'
find . -maxdepth 1 -type f -mtime 1 -printf '%f\n'
find . -maxdepth 1 -type f -mtime +1 -printf '%f\n'
find . -maxdepth 1 -type f -mtime +0 -printf '%f\n'
```

输出依次只有：

```text
30-minutes-old

25-hours-old

49-hours-old
25-hours-old
49-hours-old
```

空行是我特意留的分组，不是漏字。判定标准也简单：`-mtime 0` 只命中 30 分钟的文件；`-mtime 1` 只命中 25 小时的文件；`-mtime +1` 只命中 49 小时的文件；`-mtime +0` 命中后两个。

### 它到底怎么算“几天”

`-mtime 1` 的意思是“最后修改时间距现在恰好一个完整的 24 小时”，也就是年龄落在 24 小时到不足 48 小时之间。`-mtime +1` 则是超过一个完整周期，实际至少 48 小时。别把它当“昨天零点到现在”，它只看滚动的 24 小时。

失败现象也很典型：想清过去 24 小时以前的东西，却写了 `-mtime -1`，结果只捞到不足 24 小时的文件；想清至少一天以前的东西，`-mtime +0` 才是对的。我的样本上它准确返回了 25 小时和 49 小时两个文件。

如果真要按日历取昨天，GNU find 可以直接写时间边界：

```bash
start=$(date -d 'yesterday 00:00:00' '+%Y-%m-%d %H:%M:%S')
end=$(date -d 'today 00:00:00' '+%Y-%m-%d %H:%M:%S')
find . -type f -newermt "$start" ! -newermt "$end" -printf '%p\n'
```

这样零点整到今天零点整的文件才算数。我最喜欢 `-mtime` 的地方，本来以为它只适合“大概多久以前”，加上时间边界后居然能精确到一段日历时间；这次踩坑倒让我把它看顺眼了。
