---
title: '别让两个定时任务同时踩进同一扇门'
date: 2026-11-18 19:18:56
tags: [Linux, flock, 定时任务, Shell, 教程]
categories: 折腾
---

今天晚上我又碰上了那个很不起眼的毛病：上一轮还没收尾，下一轮已经推门进来。博客仓库还好，最多多写一篇；可要是两个任务同时改同一个文件，轻则把对方覆盖掉，重则把半截内容推上线。它不报错，甚至可能“看起来成功”，所以我想把它压成一行能验证的锁。

Linux 自带的 `flock` 正好干这事。它不是给文件盖个章，而是在内核里给文件描述符加一把排他锁：谁先拿到，谁先跑，后来的等，或者干脆退出。

## 第 0 步：先确认机器上有它

```bash
command -v flock
flock --version
```

我这台 Debian 13 上的真实输出是：

```text
/usr/bin/flock
flock from util-linux 2.41.5
```

判定标准：第一条是绝对路径，第二条至少显示 util-linux 的版本。都没有就别急着抄后面的命令，先装 util-linux。

## 第 1 步：先造一个会占门两秒的任务

我把它写成 `demo-job.sh`：

```bash
#!/usr/bin/env bash
printf 'started at %s\n' "$(date '+%H:%M:%S')"
sleep 2
printf 'finished at %s\n' "$(date '+%H:%M:%S')"
```

给执行权限，然后拿它做对照：

```bash
chmod +x demo-job.sh
./demo-job.sh
```

输出是：

```text
started at 19:20:03
finished at 19:20:05
```

它会老老实实跑两秒。接下来才是关键：让第二个任务在门还关着的时候来敲门。

## 第 2 步：让第二份任务在同一扇门前等着

先开第一份，让它占着锁文件；再立刻试第二份：

```bash
LOCK="$PWD/job.lock"

flock "$LOCK" bash -c '
  printf "first: acquired\n"
  sleep 2
  printf "first: released\n"
' &
FIRST_PID=$!

sleep 0.2
if flock -n -x "$LOCK" bash -c 'printf "second: acquired\n"'; then
  printf 'second: got the lock\n'
else
  printf 'second: skipped, exit=%s\n' "$?"
fi

wait "$FIRST_PID"
```

我这边拿到了：

```text
first: acquired
second: skipped, exit=1
first: released
```

第二份没有进屋，`flock -n` 的退出码是 1。`-n` 就是 non-blocking：不排队，拿不到锁立刻返回，不会把下一轮定时任务吊在那里干等。

这里也有一个容易写错的细节：`flock` 的文件参数和命令参数要分开，第一种写法不能写成 `flock "$LOCK" bash -c '...'`。`-c` 后面只接一个命令字符串；像上面那样把 `bash -c` 当成真正的子命令，参数才会按位置传给脚本。我第一次把 `bash -c` 塞进引号时，`flock` 直接报了 `requires exactly one command argument`，随后因为错误分支没让第一条任务退出，第二份竟然拿到了锁——锁工具自己也会把“锁没生效”伪装成成功，所以每一步都要看真实输出。

## 第 3 步：把真正要执行的命令包进去

测试过了，发布时就用这一行：

```bash
flock -n /tmp/blog-post.lock -c './publish-blog.sh'
```

如果任务已经有人在跑，这条命令的退出码是 1；没人跑，它才执行 `publish-blog.sh`。你可以在脚本里区分：

```bash
flock -n /tmp/blog-post.lock -c './publish-blog.sh'
case "$?" in
  0) printf 'published\n' ;;
  1) printf 'previous run still active; skipped\n' ;;
  *) printf 'unexpected failure\n' >&2; exit 1 ;;
esac
```

判定标准也简单：退出码 0 才表示脚本真的执行过；退出码 1 是“上一轮还在，主动让路”，不是发布失败；其他非零值才该去看日志。

锁文件放 `/tmp` 没问题，它只是拿来对暗号的，真正被保护的是脚本要改的内容。要是任务必须等待前一轮结束，去掉 `-n`：

```bash
flock /tmp/blog-post.lock -c './publish-blog.sh'
```

但我会更偏爱 `-n`。定时任务最怕的是越积越多：上一轮三分钟，下一轮五分钟，很快旧任务还没走，新任务又排在了后面。宁可明确跳过一次，也不要让它们在队列里叠成一串。

## 第 4 步：把退出结果留下来

`flock` 成功锁住时不会替我制造日志，所以我会让被执行的脚本自己写一行：

```bash
flock -n /tmp/blog-post.lock -c \
  './publish-blog.sh && printf "published at %s\n" "$(date --iso-8601=seconds)" >> /tmp/blog-post.log'
```

连续跑三次，再看日志：

```bash
for i in 1 2 3; do
  flock -n /tmp/blog-post.lock -c 'sleep 1'
done
cat /tmp/blog-post.log
```

判定标准不是“命令没报错”，而是日志里恰好有一条成功记录。若前一轮超过两秒，后面两次都应返回 1，日志仍只有一条——这正是互斥该有的样子。

最后记得把 `demo-job.sh` 和试验用的 `job.lock` 删掉。锁不是防火墙，它只挡住遵守同一把锁的进程；某个任务忘了加 `flock`，照样能闯进来。我今晚先把最容易撞车的那几个任务包上，再给它们各留一把不同的锁——锁名相同会互相挡，锁名不同则各走各的门。
