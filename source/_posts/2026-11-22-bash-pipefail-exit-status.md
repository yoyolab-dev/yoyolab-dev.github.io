---
title: '管道里有人失败了，脚本为什么还说成功？'
date: 2026-11-22 10:18:31
tags: [Bash, Shell, 管道, 程序员工具]
categories: [技术教程]
---

晚上整理脚本时，我差点被一个很会装没事的管道骗了。左边命令明明失败了，右边命令却成功了，整条管道还返回 0。若把它放进定时任务，日志里一片安静，真正出错的那一步却溜走了。

## 1. 先把“假成功”复现出来

我先执行这条最小例子：

```bash
bash -c 'false | true; printf "plain=%s\n" "$?"'
```

实际输出是：

```text
plain=0
```

`false` 的退出码是 1，`true` 的退出码是 0；管道默认只看最后一个命令，所以整条管道被算成成功。判定标准就在这里：左边明明失败，`$?` 却是 0，这就是需要修的信号。

## 2. 让管道把失败说出来

Bash 自带 `pipefail`。我把它加进临时脚本 `pipeline-check.sh`：

```bash
#!/usr/bin/env bash
set -o pipefail

false | true
status=$?
printf 'pipeline status=%s\n' "$status"
exit "$status"
```

运行：

```bash
bash pipeline-check.sh
echo "script exit=$?"
```

实际结果是：

```text
pipeline status=1
script exit=1
```

现在失败没有被藏住了。`set -o pipefail` 只负责让管道返回最靠右的那个非零退出码；想让脚本在这一步立刻停下，可以再配合 `set -e`。如果使用 `set -euo pipefail`，通常要放在脚本靠前的位置。

这里还有个容易踩的小坑：`$?` 只能代表刚刚执行的命令，所以要立刻用 `status=$?` 保存下来。晚一步再读，它可能已经变成下一条命令的退出码了。

## 3. 管道有三段，就逐段查

如果我不只想知道“失败了”，还想知道是哪一段失败，可以看 Bash 的 `PIPESTATUS`：

```bash
bash -c 'false | true | true; printf "statuses=%s,%s,%s\n" "${PIPESTATUS[0]}" "${PIPESTATUS[1]}" "${PIPESTATUS[2]}"'
```

实际输出：

```text
statuses=1,0,0
```

三个数字依次对应三个命令：第一段失败，后两段成功。这个数组同样要马上读取，下一条命令执行后它就会被刷新。

实际改脚本时，我会先保留 `pipefail`，再用 `PIPESTATUS` 定位坏环节。若系统自带的 `/bin/sh` 不支持这个选项，症状是 `set: pipefail: not found`；把脚本解释器改成 `#!/usr/bin/env bash`，或明确用 `bash script.sh` 运行即可。

我以前只看管道末尾的退出码，像只检查一串珠子的最后一颗。现在会多问一句：前面那几颗有没有悄悄掉了？这种小检查，常常就是定时任务从“看起来成功”变成“真的成功”的分界线。
