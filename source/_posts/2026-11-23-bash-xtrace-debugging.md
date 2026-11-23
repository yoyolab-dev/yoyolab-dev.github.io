---
title: '脚本只报一行错时，让 Bash 把走过的路摊开'
date: 2026-11-23 21:18:33
tags: [Bash, Shell, 调试, 程序员工具]
categories: [技术教程]
---

昨晚我又跟一个只肯说“失败了”的脚本较劲。它屏幕上只留了一行错误，前面翻了多少个文件、算出了什么，全都看不见。后来我给 Bash 开了追踪，那条藏起来的路一下就摊在屏幕上了。

### 1. 先做一个能稳定复现的小脚本

在 `~/demo` 里新建 `check.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

count=$(printf 'alpha\nbeta\n' | wc -l)
echo "count=$count"
test "$count" -eq 1
echo done
```

运行：

```bash
bash ~/demo/check.sh
echo "exit=$?"
```

我这里实际输出是：

```text
count=2
exit=1
```

`done` 没有出现，因为 `test 2 -eq 1` 返回了 1，而 `set -e` 让脚本当场停下。判定标准也在这里：退出码非 0，且最后一条 `echo` 没有输出，这就是一个能重复追查的失败。

### 2. 用 `bash -x` 看见每条命令

不改编脚本，先从外面打开追踪：

```bash
bash -x ~/demo/check.sh 2>&1
```

这次能看到完整过程：

```text
+ set -euo pipefail
++ printf 'alpha\nbeta\n'
++ wc -l
+ count=2
+ echo count=2
count=2
+ test 2 -eq 1
exit=1
```

双层 `++` 表示命令替换里的内部命令。真正把脚本截停的是最后一行的 `test 2 -eq 1`，不是前面的 `wc`。普通输出 `count=2` 与追踪行混在一起也没关系，排查阶段这样看最省事。

如果一行行读源码位置还不够，我还会给追踪加行号：

```bash
PS4='+[${LINENO}] ' bash -x ~/demo/check.sh 2>&1
```

我这次得到的关键几行是：

```text
+[3] printf 'alpha\nbeta\n'
+[3] wc -l
+[3] count=2
+[5] test 2 -eq 1
```

`5` 直接指向脚本第 5 行。要是提示 `PS4: parameter null or not valid`，多半不是脚本坏了，而是命令交给了不支持这种写法的 `/bin/sh`；改用 `bash check.sh` 即可。

### 3. 把正常输出和追踪日志分开

追踪内容会变多，我不想它污染程序真正的输出，于是改写成：

```bash
bash -x ~/demo/check.sh >result.txt 2>trace.log
echo "exit=$?"
wc -c result.txt trace.log
wc -l trace.log
```

实测得到：

```text
exit=1
8 result.txt
95 trace.log
6 trace.log
```

`result.txt` 只有 8 字节，里面就是 `count=2`；追踪日志有 95 字节、6 行。若文件明明存在却得到 `No such file or directory`，先检查重定向目标所在目录是否创建；`2>` 写错时，Bash 会在运行脚本前就报错。

现在可以只筛可疑命令：

```bash
grep -F '+ test' ~/demo/trace.log
```

输出为：

```text
+ test 2 -eq 1
```

看到它却没有别的线索时，就从追踪日志末尾往前找；判断通过的标准不是“日志里有 `+`”，而是**能指出具体失败命令、实际参数和退出码**。

### 4. 追踪时别把秘密一起追出来

`bash -x` 会把赋值后的值也打印出来。令牌这类内容不该出现在日志里，我会在附近临时关掉追踪：

```bash
set +x
token="$TOKEN"
printf 'value=%s\n' "$token"
set -x
```

用 `TOKEN=TOKEN_VALUE bash -x` 实测，追踪里只有 `+ set -euo pipefail` 和 `+ set +x`，没有令牌内容，程序自己的 `value=TOKEN_VALUE` 仍会正常输出。注意 `set +x` 必须放在秘密赋值之前；已经打印出来的东西，事后关追踪可救不回来。

我以前遇到 Bash 脚本失败，只盯着最后一行报错来回猜。现在会先让它把走过的路说出来，再把两条输出分流：正常结果看屏幕，执行轨迹单独留档。那种终于看见“它到底在第几步拐错弯”的踏实感，确实很值钱。
