---
title: '文件到底有没有变？别让大小和修改时间骗了你'
date: 2026-11-23 19:18:34
tags:
  - Linux
  - 文件校验
categories:
  - 技术笔记
---

前几天我整理两份目录时，发现一个很会骗人的情况：两个文件的名字一样，大小一样，修改时间也一模一样，看起来就像完全没变。可真正打开内容，又是两回事。

这类情况不常见，却值得单独拿出来说。因为“复制成功”“备份完成”都不能只看大小和时间，我更愿意再问一句：**文件内容真的相同吗？**

### 1. 先做一个大小和时间都相同的文件

在 `~/demo` 里准备两个文本文件：

```bash
mkdir -p ~/demo
printf 'alpha\n' > ~/demo/work.txt
cp -p ~/demo/work.txt ~/demo/backup.txt
printf 'bravo\n' > ~/demo/work.txt
touch -r ~/demo/backup.txt ~/demo/work.txt
```

`touch -r` 让 `work.txt` 沿用备份文件的时间戳。随后用 `stat` 验收：

```bash
stat -c '%n | size=%s | mtime=%y' \
  ~/demo/work.txt ~/demo/backup.txt
```

我这里得到的两行是：

```text
.../work.txt | size=6 | mtime=2026-11-23 19:20:38.867311427 +0800
.../backup.txt | size=6 | mtime=2026-11-23 19:20:38.867311427 +0800
```

大小都是 6 字节，修改时间也一致。若同步工具只看这两个字段，它会认为无需传输。

### 2. 用 SHA-256 看内容指纹

```bash
sha256sum ~/demo/work.txt ~/demo/backup.txt
```

实测输出：

```text
5da8f23decf397b13f4f55b6fb8a61936238bfe08ed9d901132974f1beccc45c  .../work.txt
b6a98d9ce9a2d9149288fa3df42d377c3e42737afdcdaf714e33c0a100b51060  .../backup.txt
```

两个 64 位十六进制摘要不同，说明内容不同。判定标准很简单：摘要相同才可认为内容一致；摘要不同，就不要再被大小和修改时间骗了。

如果命令报 `No such file or directory`，先检查路径拼写；需要批量比较时，可以在目录内执行：

```bash
find ~/demo -type f -exec sha256sum {} +
```

### 3. 要精确到“哪里不一样”，用 cmp 和 diff

`cmp` 专门逐字节比较：

```bash
cmp -s ~/demo/work.txt ~/demo/backup.txt
echo "cmp exit=$?"
```

这次得到 `cmp exit=1`。它不是系统故障，而是标准的“内容不同”：`0` 表示相同，`1` 表示不同，`2` 才表示文件缺失或读取出错。

文本文件再接 `diff`，差异会直接摊开：

```bash
diff -u ~/demo/backup.txt ~/demo/work.txt
```

输出里只有真正变了的那一行：

```diff
-alpha
+bravo
```

如果 `diff` 没显示内容，先别急着判定相同；二进制文件可以改用 `cmp -l` 看首个不同字节的位置。

我以前容易把“文件存在”“大小没变”当成复制正确，现在会顺手多算一次摘要。大文件的确会多读一遍磁盘，但比起悄悄留下两份同貌不同的内容，这点成本很值得。
