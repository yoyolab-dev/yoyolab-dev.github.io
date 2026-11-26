---
title: '硬链接和软链接不是一回事：我删掉了原文件'
date: 2026-11-26 15:27:43
tags:
  - Linux
  - 文件系统
  - 命令行
categories:
  - 技术
---

今天我差点把两个看起来差不多的东西混在一起：硬链接和软链接。它们都能用另一个名字打开同一个文件，可我把原文件删掉以后，一个稳稳活着，另一个立刻变成了断掉的箭头。于是我在临时目录里把这件事完整走了一遍。

先准备三个名字：

```bash
mkdir -p ~/hardlink-demo && cd ~/hardlink-demo
printf 'hello hard\n' > original.txt
ln original.txt hard.txt
ln -s original.txt soft.txt
```

`ln` 不加 `-s` 建硬链接，`ln -s` 建符号链接。接着用 `stat` 看三者的 inode 和链接数：

```bash
stat -c '%i | links=%h | size=%s | %F | %n' original.txt hard.txt soft.txt
```

我这里得到：

```text
5587803223974817388 | links=2 | size=11 | regular file | original.txt
5587803223974817388 | links=2 | size=11 | regular file | hard.txt
10392143322504168520 | links=1 | size=12 | symbolic link | soft.txt
```

判定标准很直接：`original.txt` 与 `hard.txt` 的 inode 完全相同，链接数都是 2；`soft.txt` 有自己的 inode，只是记住了一个路径。软链接的 `size=12` 是路径 `original.txt` 的长度，不是里面文字的长度。

现在从原文件写入新内容：

```bash
printf 'changed through original\n' > original.txt
cat original.txt hard.txt soft.txt
```

三个名字都读到了 `changed through original`。但这种相似很会骗人。接下来删掉原文件：

```bash
rm original.txt
find . -maxdepth 1 -type f -printf '%f | size=%s\n' | sort
```

输出只剩：

```text
hard.txt | size=25
```

硬链接还在，因为 `hard.txt` 本身就是那个 inode 的另一个目录项。软链接则只剩一条“原路”：

```bash
readlink soft.txt
test -e soft.txt; echo $?
test -L soft.txt; echo $?
```

依次是 `original.txt`、`1`、`0`。也就是 `test -e` 判定目标不存在，`test -L` 却确认它仍是符号链接。修法不是删掉这个坏箭头，而是让目标重新存在，或用 `rm soft.txt` 清掉它。

再补一个查同源文件的办法：

```bash
printf 'same file again\n' > one.txt
ln one.txt two.txt
find . -maxdepth 1 -samefile one.txt -printf '%f | inode=%i | links=%n\n' | sort
```

结果把 `one.txt` 和 `two.txt` 一起列了出来，inode 都是 `3777539259534869537`，链接数为 2。删除 `one.txt` 后，`two.txt` 仍能读到 `same file again`。

硬链接也有两个明确边界。我尝试给目录建硬链接，命令以退出码 1 结束，并提示 `hard link not allowed for directory`；再从 `/dev/shm` 给当前文件系统建链接，则报 `Invalid cross-device link`。普通用户不能给目录建硬链接，硬链接也不能跨文件系统；遇到这两种报错，不是命令拼错，而是文件系统不答应。

我原来以为“链接”只是文件外面的两张便签，这次才看清：硬链接更像同一扇门上的两个门牌，软链接则是一张写着地址的纸条。地址可以失效，门牌却可以都还在。
