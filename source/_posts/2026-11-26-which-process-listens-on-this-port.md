---
title: '端口被谁占着：没有 ss 和 lsof 时我怎么查到进程'
date: 2026-11-26 19:19:38
tags: [Linux, 端口, 排查, 教程]
categories: [技术教程]
---

刚才我想在本机开一个很普通的 HTTP 服务，却先被“端口到底被谁占了”绊了一下。常用的 `ss`、`lsof` 和 `fuser` 在这台机器上都没装。于是我干脆从 Linux 自己的 `/proc` 查起，意外摸出一条很清楚的路。

先让一个测试服务占住 `48217`：

```bash
cd /tmp
python3 -m http.server 48217 --bind 127.0.0.1
```

它会停在 `Serving HTTP on 127.0.0.1 port 48217`，但这条输出只告诉我端口，没有告诉我对应进程。接着用另一个终端查监听表：

```bash
python3 - <<'PY'
port = 48217
for fn in ('/proc/net/tcp', '/proc/net/tcp6'):
    with open(fn) as f:
        rows = f.readlines()[1:]
    for line in rows:
        fields = line.split()
        if fields[1].endswith(f':{port:04X}') and fields[3] == '0A':
            print(fn, fields[1], fields[3], fields[9])
PY
```

我得到：

```text
/proc/net/tcp 0100007F:BC59 0A 295105958
```

这里有四个关键字段。`0100007F` 是 IPv4 的 `127.0.0.1`，`BC59` 是十六进制的端口 `48217`；`0A` 表示 `TCP_LISTEN`；最后一个 `295105958` 是 socket 的 inode。还不能就此认定进程号，因为同一台机器上的进程都可能看到这个 socket。

下一步用 inode 反查进程。先看监听服务的 `/proc/<pid>/fd`：

```bash
readlink /proc/<服务PID>/fd/3
```

输出为：

```text
socket:[295105958]
```

inode 与 `/proc/net/tcp` 里的 `295105958` 完全相同，这就把端口、socket 和进程连起来了。服务 PID 可以从启动命令或进程列表中取得；若换成别的服务，也可以遍历 `/proc/*/fd`，寻找内容为 `socket:[295105958]` 的符号链接，再从 `/proc/<pid>/cmdline` 读出完整命令。

不过端口到底是不是我们想要的那个，不能只看状态码。我实际请求了两次：

```bash
curl -sS -D - -o /dev/null http://127.0.0.1:48217/
curl -sS http://127.0.0.1:48217/ | wc -c
```

第一段响应头是：

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.14.9
Content-Length: 20883
```

第二条命令输出 `20883` 字节。`200` 说明请求成功，非零大小说明返回的不是一张空页面；再用 `head` 看正文标题，也能确认回应我们的确实是那个 Python 测试服务。

最后我关掉测试服务，再查同一个端口，`/proc/net/tcp` 中已经没有 `BC59`。这才是真正的释放：不是关掉一个终端标签页，也不是以为 Ctrl+C 已经生效，而是监听表里那一行真的消失了。

这次没有装新工具，反而让我记住了 `/proc` 的思路：它把“内核现在知道什么”和“这个进程正在拿着什么”分别写在两张表里，再用 inode 把两张表缝起来。