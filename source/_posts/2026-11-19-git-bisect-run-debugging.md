---
title: '别再手翻提交了，让 git bisect run 自己抓第一个坏提交'
date: 2026-11-19 10:18:31
tags:
  - Git
  - 调试
  - 程序员工具
categories:
  - 技术
---

今天上午改一个小脚本，连着提交了几次以后，页面突然白了。我盯着最近十几次提交，越看越觉得可疑的不是哪一条，而是到底该从哪一条开始怀疑。以前我会从最新提交一路往回点，运气好二十分钟找到，运气不好能把眼睛看酸。

这次我没有手翻，而是让 `git bisect run` 替我做。它的工作方式很像翻页查字典：先找一个确定正常的历史点，再找一个确定坏掉的历史点，中间的提交由 Git 二分，交给一个脚本判断“这次好不好”。

我先在临时目录做了一个只有五个提交的练习仓库。`check.sh` 做的事情很简单：运行 `python3 app.py`，语法没错就返回 0，有错就返回非 0。

```bash
#!/usr/bin/env bash
python3 app.py >/dev/null 2>&1
```

接着准备两个边界。第一个提交能运行，最后一个提交已经坏了：

```bash
git bisect start
git bisect bad HEAD
git bisect good a89e8cb
```

这里的 `good` 不是“我觉得应该没问题”，而是你真的跑过、确认它没坏。边界选错，结论就会跟着错，所以我先把完整提交列看了一遍：

```bash
git log --oneline --reverse
```

输出的五条里，第一条 `base: build passes` 是好的，最后一条 `change: add another line` 是坏的。确认完再开始二分：

```bash
git bisect run ./check.sh
```

这次 Git 先测中间的 `change: introduce syntax error`，脚本返回非 0，于是它往前退；再测 `change: build still passes`，脚本返回 0，于是它往前追。最终输出是：

```text
137845b7c97ed48a608ae14f35d5931496aa9d99 is the first bad commit
commit 137845b7c97ed48a608ae14f35d5931496aa9d99
...
change: introduce syntax error
bisect found first bad commit
```

五个提交只实际运行脚本两次，就找到了那条真正把括号漏掉的提交。`git bisect log` 还能把整个判断过程记下来，`git bisect reset` 则会退出二分状态，回到原来的分支。

真正拿来查线上问题时，脚本要换成能准确复现故障的命令。比如构建失败就用项目的构建命令，测试挂了就跑那条测试。判定标准只有一条：正常时退出码为 0，异常时退出码非 0。若脚本永远返回 0，Git 会误以为所有提交都正常；若它始终返回非 0，则会把最早的提交也判成坏的。脚本还必须清理临时文件，否则前一次测试留下的东西会污染下一次结果。

我觉得 `git bisect run` 最顺手的地方，不是它比人快，而是它把“我记得好像是那次改坏的”变成了可复查的提交号。查到以后我仍会打开 diff 看一眼，只是这次不用再从几十页日志里捞那一行括号了。
