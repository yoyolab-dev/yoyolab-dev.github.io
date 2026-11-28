---
title: '把改到一半的活收进抽屉：git stash 的用法和一次翻车'
date: 2026-11-28 21:55:18
tags:
  - Git
  - 命令行
  - 工作流
categories:
  - 技术
---

手头的活改到一半，突然要切去别的分支改个 bug，我最怕的就是把改动弄丢。于是每次都先 `git stash` 一下，把工作区变干净，再去忙别的。这次我把这条路径整个走了一遍，顺便被它教育了一次——`stash push` 默认根本不收新文件，差点让我以为存好了。

先造一个能复现的场景。下面所有命令都在空目录里执行，我的环境是 git 2.47.3：

```bash
mkdir stash-demo && cd stash-demo
git init -q -b main .
printf 'line1\nline2\n' > a.txt
git add a.txt
git commit -qm init
printf 'line1\nline2\nchanged\n' > a.txt
printf 'brand new\n' > b.txt
git status --short
```

输出两行，前面的空格很重要：

```text
 M a.txt
?? b.txt
```

` M` 表示已跟踪文件被改了但没进暂存区，`??` 表示这是个 Git 还不认识的新文件。

第一步，把活收进抽屉：

```bash
git stash push -m 'wip: 改 a.txt'
git status --short
git stash list
```

真实输出：

```text
Saved working directory and index state On main: wip: 改 a.txt
?? b.txt
stash@{0}: On main: wip: 改 a.txt
```

注意 `b.txt` 还站在原地。`stash push` 默认只收已跟踪文件的改动，新文件它看都不看——这是它给我的第一个教训。想连未跟踪文件一起收，得明说：

```bash
git stash push -u -m 'wip2: 带上 b.txt'
git status --short
git stash list
```

这次 `git status --short` 什么都没输出，工作区干净了；`git stash list` 变成两条：

```text
stash@{0}: On main: wip2: 带上 b.txt
stash@{1}: On main: wip: 改 a.txt
```

第二条里其实只有 `a.txt` 的改动，而 `stash@{0}` 里有 `b.txt`。想看清某条到底装了什么，`git stash show --stat` 就够，但有个坑：查含未跟踪文件的那条，不加 `-u` 会什么都看不见。

```bash
git stash show --stat 'stash@{0}'
git stash show -u --stat 'stash@{0}'
```

第一条输出是空的，第二条才是：

```text
 b.txt | 1 +
 1 file changed, 1 insertion(+)
```

接下来取回。编号不是固定的，`stash@{0}` 永远是最新的那条，我要的是旧的那条：

```bash
git stash pop 'stash@{1}'
cat a.txt
git stash list
```

`cat a.txt` 打出 `changed`，说明改动回来了；`git stash list` 里 `stash@{1}` 消失了，只剩 `stash@{0}`——`pop` 成功后会顺手把这条删掉，所以编号也跟着往前挪了一位。

然后是那次翻车。我在工作区又改了同一个 `a.txt`，再执行 `git stash pop`：

```text
error: Your local changes to the following files would be overwritten by merge:
	a.txt
Please commit your changes or stash them before you merge.
Aborting
The stash entry is kept in case you need it again.
```

退出码是 1，`a.txt` 还是我刚写的那份，而 stash 那条**没有丢**——Git 特意写了 "The stash entry is kept"，这是它给的安全网。修法很简单：先把手上的改动也收起来，再去 pop。

```bash
git stash push -m 'wip: worktree 版'
git stash list
git stash pop 'stash@{1}'
```

这次 `a.txt` 变成了 stash 里那份，退出码 0。

剩下三个小工具我也试了。`git stash apply` 和 `pop` 一样取回内容，但**不删**这条，适合想反复对比的时候；`git stash branch feature` 更干脆，直接从 stash 开一条新分支，出来时已经在 `feature` 上、文件也是 stash 里那份，而 stash 列表被清空了；`git stash drop` 则是明确丢弃一条。

最后一个小细节：暂存区能不能一起带回来。

```bash
git add a.txt        # 暂存后再 stash
git stash pop        # 普通 pop
git status --short   #  M a.txt
git stash pop --index
git status --short   # M  a.txt
```

普通 `pop` 之后是 ` M`（改动还在，但没进暂存区），加上 `--index` 才是 `M `（改动在暂存区里）。我那次差点以为暂存状态丢了，其实是默认不还原。

抽屉最妙的地方在于它不占脑子：你不用想这条存了什么，`git stash list` 会告诉你；你不用怕取错，`pop` 失败时它会把东西留在原地。我现在改代码改到一半要跳走，第一件事就是 `git stash push -u -m 'wip: 在干嘛'`，那个 `-u` 是我今天用一次翻车换来的。

![深夜的桌前，她伸手拉开那只放着半成品纸张的抽屉](/images/git-stash-drawer.jpg)
