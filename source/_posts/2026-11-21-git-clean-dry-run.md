---
title: '先看 Git 准备删什么，再把钥匙递出去'
date: 2026-11-21 10:18:29
tags: [Git, Shell, 程序员工具, 清理]
categories: [技术教程]
---

今天整理博客仓库时，我突然很在意一件事：`git clean` 真要动手之前，我能不能先看见它准备删掉哪些东西？

以前我只在确信目录完全干净时用 `git clean -fd`。可人的 certainty 很不可靠，一枚误放进来的 `.env`、一个还没提交的草稿，都可能被 `-x` 扫走。于是我重新搭了一个小仓库，把安全开关一个个试过去。

## 1. 先造一个“看起来很干净”的仓库

我在临时目录里放了三类东西：已经提交的 `notes.txt` 和 `config/settings.json`，未跟踪的 `report.md`，以及被 `.gitignore` 忽略的 `.env` 和 `build/`。`.gitignore` 只有两行：

```gitignore
.env
build/
```

先看普通状态：

```console
$ git status --short
?? report.md
```

这里已经露出一个容易误会的点：`.env` 和 `build/` 根本没显示，因为 Git 默认不汇报被忽略的文件。也就是说，“`git status` 看起来没有秘密”，不等于清理时不会碰到它。

## 2. `-n` 是先演一遍，不是真删

我先执行最保守的预览：

```console
$ git clean -nd
Would remove report.md
```

`-n` 是 dry run，只列清单。执行前后，工作树里的文件数都是 **6**，用 `diff` 比较清单也是 **0 行差异**。判定标准很简单：预览命令里只要出现 `Would remove`，它就没有动文件。

接着我把忽略项也纳入视野：

```console
$ git clean -ndx
Would remove .env
Would remove build/
Would remove report.md
```

这里 `-x` 的作用是把被忽略的文件也算进清单，`-d` 则把目录纳入清理范围。它们和 `-n` 组合起来，正适合我这种“不敢先删、只想先看”的脾气。

## 3. 真正清理时，我会把范围写窄

如果只想处理一个草稿目录，可以先用路径限定预览：

```console
$ git clean -nd -- report.md build/
Would remove report.md
```

等等，这个结果有点怪：`build/` 没出现在定向预览里，因为 `build/` 已经被 `.gitignore` 忽略，普通模式不会碰它；想连它一起看，得明确使用 `-x`：

```console
$ git clean -ndx -- report.md build/
Would remove build/
Would remove report.md
```

确认清单与预期完全一致后，才去掉 `-n`：

```console
$ git clean -fdx -- report.md build/
```

`-f` 是真正执行。若只想清未跟踪文件、不碰忽略项，就用 `git clean -fd -- report.md`；若目标可能包含重要文件，先把它移出仓库或提交，不要期待 Git 替我恢复。

我最喜欢这个小实验的地方，是它把“我以为仓库里有什么”变成了可核对的文件数、清单和 `diff`。删文件不可逆，但预览可以随便做；先看清那几行 `Would remove`，手再伸向 `-f`，心里踏实得多。
