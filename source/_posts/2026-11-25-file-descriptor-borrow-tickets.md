---
title: 'Too many open files：文件描述符其实是一张借阅证'
date: 2026-11-25 19:19:19
tags:
  - Linux
  - Python
  - 文件描述符
categories:
  - 技术笔记
---

前几天我遇到一条很短的报错：`OSError: [Errno 24] Too many open files`。第一眼很像机器里文件太多，其实不是。它说的是：当前进程能同时打开的文件描述符（file descriptor，FD）用完了。

## 先看看进程手里有多少张“借阅证”

在 Linux 上，每个进程的 FD 都在 `/proc/<PID>/fd/` 里。先查当前 shell 的 PID 和打开数量：

```bash
echo $$
ls -l /proc/$$/fd
```

我这次得到 PID `541621`，FD 目录里有 `4` 个条目：0、1、2 是标准输入、输出、错误，另一个是 Python 临时执行时打开的目录。普通 shell 一般也只有这几项。

每张“借阅证”对应的不是文件名，而是一个整数编号。`0` 通常指 stdin，`1` 指 stdout，`2` 指 stderr；文件、socket、pipe 也都占一个编号。`ls -l` 里的箭头才是它当前指向的东西，例如 `socket:[286821386]`。

## 故意把借阅证用光

下面这段 Python 每次打开 `/dev/null`，却故意不关闭：

```python
import os
import resource

soft, hard = resource.getrlimit(resource.RLIMIT_NOFILE)
print(f"soft_limit={soft} hard_limit={hard}")

opened = []
try:
    for _ in range(1000):
        opened.append(os.open("/dev/null", os.O_RDONLY))
except OSError as exc:
    print(f"opened={len(opened)} errno={exc.errno} error={exc.strerror}")
```

正常环境下，软限制是 `4096`，硬限制是 `524288`。前 1000 次不会报错，因为限额还远。为了让现象出现得更快，我在子进程里临时把软限制降到 64：

```bash
bash -c 'ulimit -n 64; python3 fd_demo.py'
```

实际输出是：

```text
soft_limit=64 hard_limit=64
opened=61 errno=24 error=Too many open files
```

为什么只开了 61 个？因为标准输入、输出、错误已经占了 0、1、2，Python 自己还用了几个目录或管道，最后一次 `open()` 想再拿一张证时，系统发现 64 张已经全部借完，于是返回 errno 24。这个数字不是固定值，换成别的运行环境会变；判断标准是错误码 `24` 和 `Too many open files` 同时出现。

## 找到是谁没还

继续看同一个进程的 `/proc/<PID>/fd/`。刚才一共创建了 1001 个 `/dev/null` FD，而 `/proc` 目录自己也会占临时 FD，所以我这次统计到 1004 项，其中 1001 项都指向 `/dev/null`：

```text
fd_entries_while_open=1004
fd_target_count=1001 target=/dev/null
fd_target_count=2 target=pipe:[286821386]
```

这比一句“疑似 FD 泄漏”有用多了：不是文件种类太杂，而是同一个 `/dev/null` 被借了 1001 次。真实项目里，指向的也可能是 socket、数据库连接或日志文件；按目标聚合后，异常集中在哪里一眼就清楚。

低限额实验还有个细节：`limit=64` 时，连 `os.listdir('/proc/<PID>/fd')` 都可能因为申请新 FD 而失败，得到同一个 errno 24。此时先把已打开的 FD 关掉，再重新读取目录，条数会从满额状态回落。

## 修法不是调大限额，而是把证还掉

把循环里的 `os.close(fd)` 补上，再执行相同实验，FD 目录只剩 4 项，之前那 1001 个 `/dev/null` 全部消失。Python 里优先用上下文管理器，它会在退出 `with` 时自动关闭：

```python
with open("/dev/null", "rb") as fp:
    fp.read(1)
```

如果业务确实需要同时打开很多文件，先查清是谁在积攒；实在需要更高额度，再在启动脚本里调整 `ulimit -n`。但我只是把限额从 64 调高，没有修那个忘记关闭的循环，下一次照样会漏光。查 `/proc/<PID>/fd` 很有用，可修法最后还是要落回“谁拿了、谁来还”。
