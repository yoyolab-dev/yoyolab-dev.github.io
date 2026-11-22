---
title: '写不进新文件时，先把 du、df 和 inode 分开问'
date: 2026-11-22 15:18:48
tags:
  - Linux
  - 磁盘
  - 故障排查
categories:
  - 技术
---

今天下午我想弄明白一种很拧巴的报错：程序说磁盘满了，可 `df -h` 明明白白写着还剩 1.2T。只盯着百分比很容易绕进去，后来我才想起：`df`、`du` 和 inode 根本不是同一本账。

## 先搭一个能复现的小目录

我不想拿真实数据乱扫，就在临时目录里做了一组样本。以下路径统一写成 `~/du-lab`，实际命令放在当前目录执行即可。

```bash
mkdir -p ~/du-lab/{logs/archive,photos,notes}
dd if=/dev/zero of=~/du-lab/logs/app.log bs=1M count=12 conv=fsync
dd if=/dev/zero of=~/du-lab/logs/archive/old.log bs=1M count=8 conv=fsync
dd if=/dev/zero of=~/du-lab/photos/sunset.raw bs=1M count=5 conv=fsync
dd if=/dev/zero of=~/du-lab/notes/today.txt bs=1M count=1 conv=fsync
```

先看目录实际占了多少：

```console
$ du -sh ~/du-lab
26M    /home/user/du-lab
```

`du` 沿着目录把文件加起来，结果是 26M。再按第一层拆开：

```console
$ du -h --max-depth=1 ~/du-lab | sort -h
1.0M    /home/user/du-lab/notes
5.0M    /home/user/du-lab/photos
20M    /home/user/du-lab/logs
26M    /home/user/du-lab
```

这里 `logs` 是 20M，正好等于 12M 的 `app.log` 加 8M 的 `old.log`。若结果对不上，就继续往下钻：

```bash
du -h --max-depth=2 ~/du-lab | sort -h
```

## 找最大的文件，别只看文件夹名字

下面这条命令只列大于 5M 的普通文件，并把真实字节数放在前面：

```bash
find ~/du-lab -type f -size +5M -printf '%10s  %p\n' | sort -nr
```

我这边的输出是：

```console
  12582912  /home/user/du-lab/logs/app.log
   8388608  /home/user/du-lab/logs/archive/old.log
```

12582912 正好是 12×1024×1024。看到这里，大头是谁已经不用猜了。`-size +5M` 用的是 MiB，不是 MB；要找 5,000,000 字节，可以改用 `-size +5000000c`。

## `du` 小，不代表磁盘没满

`df` 查的是整个文件系统，`du` 查的是目录里能数到的文件。两者差很多时，我一般先查这两件事。

第一件是已经删除、却仍被进程打开的文件。名字从目录里消失了，进程没关句柄，数据块就还没释放：

```bash
find /proc/[0-9]*/fd -lname '* (deleted)' -printf '%p -> %l\n' 2>/dev/null
```

普通用户只能看到自己有权查看的进程。如果输出里有真实的 `(deleted)` 文件，先记下 `/proc/<PID>/fd/...` 里的 PID，确认进程用途后再决定重启或关闭它；别看到文件就乱杀。

第二件是 inode 用光了。数据块还空着，但文件名额已经用完时，新文件同样创建不了：

```console
$ df -i /
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
1:2             586053248 860036 585193212    1% /
```

我这台机器是 1%，所以 inode 不是这次的原因。真正该查的是某个目录里塞了多少小文件：

```bash
du --inodes -h --max-depth=1 /path/to/check | sort -h
```

我的样本目录一共 5 个文件，计数结果是 10，因为目录本身也占 inode。

## 还有一个容易骗人的东西

我顺手做了个 2G 的稀疏文件：

```bash
truncate -s 2G ~/du-lab/sparse.bin
ls -lh ~/du-lab/sparse.bin
du -h ~/du-lab/sparse.bin
du -h --apparent-size ~/du-lab/sparse.bin
```

结果是：文件“看起来”有 2.0G，`du` 却报 0，`du --apparent-size` 才报 2.0G。因为前面的内容还没真正写盘。

所以排查时我会按这个顺序走：先用 `df -h` 看分区，再用 `df -i` 看 inode；接着用 `du --max-depth=1` 找目录，用 `find ... -size` 找大文件；最后再查 deleted-open 句柄。哪一步数字对不上，就停在那一层继续拆。最容易误判的已经不是“磁盘满”，而是拿 `df` 的结果去猜 `du` 的结果。
