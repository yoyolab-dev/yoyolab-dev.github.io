---
title: '备份不是复制：restic 0.20.0 让我先回答“还能恢复吗”'
date: 2026-11-23 15:18:41
tags:
  - restic
  - 备份
  - 命令行
categories:
  - 技术笔记
---

前几天我整理旧文件，翻到一份只复制过一次的备份。它安静地躺在那里，像一颗没拆开的糖：看着完整，真出事时却未必能兑现承诺。后来我拿 restic 0.20.0 做了一个很小的演练，第一次觉得“备份”不是按下复制键，而是能亲手把文件捞回来。

restic 0.20.0 在 11 月 19 日发布。这个版本有两个安全修复让我格外在意：恢复时若无法解析用户名和组名，不再把文件悄悄改成 UID/GID 0；遇到目录名重复或顺序不对的快照树，也会直接拒绝恢复，而不是沿着符号链接写到目标目录外面。一个怕恢复时踩空，一个怕恢复时越界，恰好都是备份最该守住的两条线。

下面我不拿自己的真数据冒险，只建一个 48 字节的测试目录，照着把“备份、检查、恢复”走一遍。

### 1. 下载后先核对 SHA-256

Linux amd64 的压缩包是 `restic_0.20.0_linux_amd64.bz2`，同一次 release 还有 `SHA256SUMS`。我实际下载后算出的 SHA-256 是：

```text
0ccd9fa7885074a24603b051e01bf9bd8b5e45d6a1791e9d9887e9e9372107f7
```

它与 release 里的 `SHA256SUMS` 对应行完全一致。随后解压得到 `restic 0.20.0 compiled with go1.27.4 on linux/amd64`。判定标准不是“下载页面能打开”，而是摘要相同、程序能打印自己的版本号。

### 2. 初始化仓库并做第一次备份

在临时目录里准备 `data/a.txt` 和 `data/sub/b.txt`，内容合计 46 字节。仓库密码只放在这次 shell 的环境变量里：

```bash
export RESTIC_PASSWORD='demo-only-password'
restic -r "$HOME/demo-repo" init
restic -r "$HOME/demo-repo" backup "$HOME/demo-data" --verbose
```

第一次没有父快照，restic 会读完全部文件。真实输出是：

```text
scan finished in 0.002s: 2 files, 46 B
Files:           2 new,     0 changed,     0 unmodified
Added to the repository: 4.193 KiB (3.276 KiB stored)
snapshot 0777b22d saved
```

“2 new、0 changed、0 unmodified”和生成了快照 ID，就算第一关通过。仓库不是源目录的复印件，它有自己的索引、加密后的数据块和快照记录。

### 3. 不改文件再备份一次

同一条 `backup` 命令再跑一次，输出变成：

```text
using parent snapshot 0777b22d
Files:           0 new,     0 changed,     2 unmodified
Added to the repository: 0 B   (0 B   stored)
snapshot 7580b7f6 saved
```

这里最值得看的不是时间，而是 `0 B stored`。第二次备份仍会生成一个快照记录，但重复的数据块不会再塞一份进去；它先找父快照，再把没变的内容认出来。

我把 `a.txt` 改成一行 `changed`，第三次结果则是：

```text
Files:           0 new,     1 changed,     1 unmodified
Data Blobs:      1 new
Added to the repository: 3.779 KiB (2.900 KiB stored)
```

一个文件变了，就只新增一个数据块，另一个文件仍被认作 `unmodified`。这比“每次都复制整个文件夹”更省空间，也比只比较文件大小更可靠。

### 4. 先验仓库，再恢复文件

备份命令成功不代表仓库里的每个块都完好，所以恢复之前要验：

```bash
restic -r "$HOME/demo-repo" check
```

我的结果：

```text
[0:00] 100.00%  3 / 3 snapshots
no errors were found
```

接着恢复到另一个目录：

```bash
restic -r "$HOME/demo-repo" restore latest --target "$HOME/restored"
```

输出为 `Summary: Restored 11 files/dirs (48 B)`。恢复目录里找到了两个文件，大小分别是 8 字节和 40 字节；我逐个读回，`a.txt` 是 `changed`，`b.txt` 仍是原来的 `beta` 串。到这里，“复制了一份”与“可以恢复”才算分开。

### 5. 删除旧快照时，别把保险丝一起拔了

restic 的 `forget` 删的是快照记录，真正没被引用的数据要再由 `prune` 清掉。测试里我删除了这组路径的 3 个快照：

```bash
restic -r "$HOME/demo-repo" forget \
  --path "$HOME/demo-data" \
  --keep-last 0 \
  --unsafe-allow-remove-all \
  --prune
```

它先列出将删除的 3 个快照，再报告 `to delete: 22 blobs / 5.296 KiB`，最后 `remaining: 0 blobs / 0 B`。重新执行 `check` 仍是 `no errors were found`。这个选项的名字很直白地写着 `unsafe`，真要用时，路径筛选和保留策略得先看清。

我最喜欢 restic 的地方，是它不把“成功备份”印在脸上就算完，而是逼着你再看一眼 `check` 和 `restore`。文件躺在硬盘上只是存在，某天真的需要时还能原样回来，才算这份备份没有白做。
