---
title: '别切走当前代码：三种办法看 Git 里的旧版本'
date: 2026-11-24 19:19:43
tags: [Git, 教程, 命令行, 程序员工具]
categories: [编程]
---

晚上翻到一个旧提交时，我第一反应总是 `git checkout` 过去，看完再切回来。可手上的活还没做完，这种来回切分支太容易把状态搅乱。于是我在一只临时 Git 仓库里把三条路都走了一遍：只看一个文件、导出整个旧版本、另外开一个目录运行旧代码。

先造两份很小的提交。第一版里的 `src/app.py` 是 `return sum(items)`，第二版改成 `return sum(item * 2 for item in items)`，另外用 `VERSION` 文件写着 `version=1` 和 `version=2`。两次提交后，当前提交是 `901d577`，上一版是 `c310347`。

### 只看旧文件：git show

```bash
git show HEAD~1:src/app.py
```

输出正好是第一版：

```python
def total(items):
    return sum(items)
```

这里 `HEAD~1` 是当前提交的父提交，冒号后面的 `src/app.py` 是仓库里的文件路径。这条命令只把旧内容打印到终端，不会改当前工作区。我执行后运行 `git status --short`，输出为空；判定标准就是：旧文件看得到，而当前改动和未跟踪文件没有变化。

如果文件被移动或改过名字，先列旧提交里的路径，免得对着猜：

```bash
git ls-tree -r --name-only HEAD~1
```

这次输出是 `VERSION`、`notes.txt`、`src/app.py` 三行。

### 拿一份完整旧版本：git archive

只想读一个文件，用上面的 `git show` 就够了；但要拿整个目录交给别的程序，`git archive` 更省事：

```bash
mkdir old-export
git archive --format=tar HEAD~1 | tar -x -C old-export
```

我在 `old-export` 里核对了三个文件，`VERSION` 是 10 bytes、`notes.txt` 是 6 bytes、`src/app.py` 是 40 bytes，大小都与旧提交一致。`tar -x` 只是把归档解到指定目录，不会把当前仓库切回旧提交。

### 想运行旧代码：git worktree

旧版本要跑测试、起服务时，我更愿意给它单独的目录：

```bash
git worktree add --detach ../app-old HEAD~1
```

输出里的 `Preparing worktree (detached HEAD c310347)` 告诉我旧提交已经准备好。此刻主目录的提交仍是 `901d577`，新目录是 `c310347`；一个目录里的 `VERSION` 是 `version=1`，另一个仍是 `version=2`。两份工作区各改各的，不会把当前分支切走。

`git worktree list` 能把关系看清楚：

```text
.../work        901d577 [master]
.../app-old     c310347 (detached HEAD)
```

用完清掉：

```bash
git worktree remove ../app-old
```

再跑一次 `git worktree list`，只剩主目录这一行，主仓库的 `git status --short` 仍然是空的。

这三种办法我各有偏爱：临时看一眼用 `git show`，要完整打包用 `git archive`，真要让旧代码跑起来就用 `git worktree`。最要紧的是别为了看一眼旧版本，把正在写的分支搬来搬去——旧版本可以看，手上的现场不用赌。
