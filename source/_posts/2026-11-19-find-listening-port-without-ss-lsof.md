---
title: '服务器没装 ss 和 lsof，怎么找出谁占了端口'
date: 2026-11-19 12:18:40
tags:
  - Linux
  - 运维
  - Python
categories:
  - 技术
---

今天我在一台很瘦的服务器上撞到一个老问题：本地服务明明起不来，我想看看 18765 端口被谁占了，可机器里既没有 `ss`，也没有 `lsof`，连 `fuser` 都找不到。装一整套网络工具当然可以，但那台机器不能随便动。

后来我发现，Linux 自己已经把答案摊在 `/proc` 下面了。顺着进程、socket 和 TCP 表三张表查一遍，不用猜，也不用先装东西。

### 1. 先确认端口真的有人听

先起一个临时服务。这里用 Python 自带的 `http.server`，只绑定本机：

```bash
python3 -m http.server 18765 --bind 127.0.0.1
```

另开一个终端，用 curl 验证。状态码和下载字节数一起看，才知道它不是“连上了但返回空页面”：

```bash
curl -sS -o /dev/null \
  -w '127.0.0.1:18765 -> %{http_code} %{size_download} bytes\n' \
  http://127.0.0.1:18765/
```

我这次得到的是：

```text
127.0.0.1:18765 -> 200 7178 bytes
```

再测一个没人监听的 18764：

```bash
curl -sS -o /dev/null \
  -w '127.0.0.1:18764 -> %{http_code}\n' \
  http://127.0.0.1:18764/
```

结果直接连接失败：

```text
curl: (7) Failed to connect to 127.0.0.1 port 18764: Could not connect to server
127.0.0.1:18764 -> 000
```

判定标准很简单：`200` 加非零字节说明服务正常，`000` 说明这个端口没有服务应答。

### 2. 从进程找到它的 socket inode

先拿服务的 PID，再用 `/proc/<PID>/fd` 找出它打开的 socket。这里的 `18765` 是十六进制 `494D`，而 `0A` 是 TCP 的 `LISTEN` 状态。

```python
import os

pid = 196183
inodes = []

for fd in os.listdir(f"/proc/{pid}/fd"):
    try:
        target = os.readlink(f"/proc/{pid}/fd/{fd}")
    except OSError:
        continue
    if target.startswith("socket:["):
        inode = target[8:-1]
        inodes.append(inode)
        print(fd, target)

for fn in ("/proc/net/tcp", "/proc/net/tcp6"):
    with open(fn) as f:
        next(f)
        for line in f:
            fields = line.split()
            if fields[9] in inodes:
                port = int(fields[1].split(":")[1], 16)
                print(fn, "port", port, "state", fields[3],
                      "inode", fields[9])
```

我实际读到的内容是：

```text
3 socket:[238115973]
/proc/net/tcp 0100007F:494D port 18765 state 0A inode 238115973
```

`0100007F` 是本机地址 `127.0.0.1`，`494D` 正是 18765。三个线索——PID、socket inode、端口——能互相印证，就不是撞上了同号的普通连接。

### 3. 确认没找错人

最后再读一次进程命令行：

```bash
tr '\0' ' ' < /proc/196183/cmdline
```

输出是：

```text
python3 -m http.server 18765 --bind 127.0.0.1
```

这一步很重要。端口号相同不代表你想找的就是它；命令行至少能证明 PID 确实和那个监听服务有关。

查完后，我用 `Ctrl+C` 停掉临时服务，再访问 18765，curl 恢复成连接失败，说明监听确实消失了。

`/proc` 不是所有系统都一样，但这种查法在 Linux 上很顺手：先用 curl 证明端口活着，再从进程的 socket inode 穿到 TCP 表，最后用命令行复核。三个小工具互相验账，基本不用靠猜。
