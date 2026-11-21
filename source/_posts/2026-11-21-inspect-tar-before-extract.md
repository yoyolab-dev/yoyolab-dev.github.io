---
title: '别急着解压：先用 tar 把压缩包里的路走一遍'
date: 2026-11-21 12:18:23
tags: [tar, Shell, 压缩包, 程序员工具]
categories: [技术教程]
---

下午整理备份文件时，我拿到一个陌生压缩包，第一反应居然是“直接解开看看”。手悬在命令上停了一下：里面有没有我不想释放的文件？解压后能不能原样还原？我不想拿自己的磁盘去赌，所以先让 `tar` 把清单摊开。

## 1. 先只看，不落地

我在临时目录做了两个很小的文件：`notes.txt` 是 6 字节，照片占 4 字节，然后打包：

```bash
mkdir -p tar-demo
cd tar-demo
printf 'hello\n' > notes.txt
printf '\xff\xd8\xff\xd9' > photo.jpg
tar -czf safe.tar.gz notes.txt photo.jpg
```

`tar -tzvf safe.tar.gz` 只列内容，不解压。我这里得到：

```text
-rw-r--r-- hermes/hermes     6 2026-11-21 12:21 notes.txt
-rw-r--r-- hermes/hermes     4 2026-11-21 12:21 photo.jpg
```

开头是权限、属主、大小、时间和名字。判定标准不是“看起来有文件”，而是文件名、类型和大小都对得上；想省掉属主和时间，可用 `tar -tzf`，只保留路径。

## 2. 解到专用目录里

清单没问题，我才让它进入一个空目录：

```bash
mkdir -p out
tar -xzf safe.tar.gz -C out
find out -type f -printf '%P %s bytes\n' | sort
```

输出是：

```text
notes.txt 6 bytes
photo.jpg 4 bytes
```

这里 `-C out` 很重要，它把释放位置限定在新目录，不让压缩包里的路径直接决定落点。解完再数文件、看大小，和清单一致才算第一步通过。

## 3. 比大小还不够，哈希要一样

我给原文件和副本算 SHA-256：

```bash
sha256sum notes.txt out/notes.txt photo.jpg out/photo.jpg
```

四条记录里，同名文件的哈希分别成对相同：

```text
5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03  notes.txt
5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03  out/notes.txt
32461d5bd1773012acef0ba15636752949bd7c2ce50f9172159d9f56cf0dd9af  photo.jpg
32461d5bd1773012acef0ba15636752949bd7c2ce50f9172159d9f56cf0dd9af  out/photo.jpg
```

如果大小相同、哈希不同，就说明内容已经被改过；若缺少某个文件或出现多出来的文件，也要先停手。这个归档本身只有 160 字节，压缩包小不代表内容少，单看体积也会骗人。

## 4. 再加一道路径检查

清单里出现绝对路径或 `..` 时，我宁可先问一句。我写了一个临时 Python 脚本，用 `tarfile.getmembers()` 取成员名，再检查 `PurePosixPath(name)` 是否为绝对路径、是否含 `..`。跑刚才的包，结果是：

```text
members=2
0        6 notes.txt
0        4 photo.jpg
UNSAFE=0
```

脚本退出码为 `0`；只要发现危险路径，就会打印 `UNSAFE=...` 并以 `1` 退出。Python 3.12 以后还可以用 `tarfile` 的过滤能力拒绝异常成员，不过自己的路径检查更直白，我看着它把每个名字过一遍才放心。

现在我碰到压缩包会多停十几秒：先列清单，再限定目录解压，最后用哈希对账。真正让我安心的不是命令多，而是每一步都有能看见的通过标准。