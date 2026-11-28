---
title: '二分找 bug：git bisect run 把 33 个提交缩到一个'
date: 2026-11-28 10:18:55
tags:
  - Git
  - 调试
  - bisect
categories:
  - 技术
---

单元测试突然红了，可我完全想不起来是哪次改动干的。翻提交记录翻到眼睛发酸，那种感觉最讨厌——明明答案就埋在里面，却只能一格一格地爬。后来我学着让 Git 替我翻：先造一个 33 个提交的演示仓库，再把判定脚本丢给它，五次判定就抓到了真凶。

![深色工作台上的一串七颗珠子，中间一颗是红的，放大镜正照着它](/images/2026-11-28-git-bisect-run-first-bad-commit.jpg)

## 第一步：造一个能自动判定的现场

```bash
mkdir -p ~/projects/bisect-demo && cd ~/projects/bisect-demo
git init && git config user.name demo && git config user.email demo@example.invalid
```

主程序 `app.sh` 只干一件事：

```bash
echo "answer=42"
```

判定脚本 `check.sh` 负责把"这次改动是好是坏"翻译成退出码：

```bash
#!/usr/bin/env bash
out="$(./app.sh)"
[ "$out" = "answer=42" ] && { echo "PASS: $out"; exit 0; }
echo "FAIL: $out"
exit 1
```

我一共提交了 33 次，其中一次写着"让输出更快（顺手改坏了答案）"，把 42 改成了 43，另外 32 次都是无关的杂提交（一次初始化，其余往 notes.txt 里追加说明）。

## 第二步：先亲手确认两端

```bash
./check.sh; echo "HEAD 处退出码 = $?"
```

当前 HEAD 处输出 `FAIL: answer=43`，退出码 1；切回最早的提交则是 `PASS: answer=42`，退出码 0。判定标准很简单：good 端脚本必须 exit 0，bad 端必须非 0，这两端搞错了后面全白搭。

## 第三步：开始二分

```bash
git bisect start
git bisect bad HEAD
git bisect good 8bc14ad
```

（8bc14ad 是我演示仓库里最早那个能跑通的提交，换成你自己仓库里对应的哈希即可。）它回了 `Bisecting: 15 revisions left to test after this (roughly 4 steps)`，说明范围已经收窄、接下来该我判了。

## 第四步：把判定交给脚本

```bash
git bisect run ./check.sh
```

真实输出（中间几次 PASS/FAIL 略）：

```text
running './check.sh'
FAIL: answer=43
Bisecting: 7 revisions left to test after this (roughly 3 steps)
...
PASS: answer=42
49bce6a is the first bad commit
```

5 次判定之后，它报出 `49bce6a`，正是那个"顺手改坏了答案"的提交。判定标准是最后一行必须出现 `is the first bad commit`，而且这个哈希要和 `git log --grep` 搜出来的提交对得上。

## 第五步：核对，然后收工

```bash
git bisect log
git show 49bce6a --stat
git bisect reset
```

这里有个小坑：跑完之后 HEAD 停在**最后一次被测的提交**上（我这边是 `51234de 补充第 15 条说明`），不是凶手本人；凶手是它打印出来的那行 first bad commit，得自己 `git show` 那个哈希。`git bisect reset` 才能回到原来的分支。

## 第六步：退出码写错，结论直接翻车

`git bisect run` 只认退出码：0 是好，1–127 是坏，125 是"这个提交没法测，请跳过"，超出 0–127 的其它数值会直接中止二分。我试了两种写法：

把"这个提交跑不起来"写成 `exit 125`，git 就把这个提交记进 `git bisect log` 当 skip 处理（`# skip: [1643fdd...]`），继续找下去，最后照样稳稳落到 `49bce6a`。换成 `exit 2`（脚本自己崩了）就惨了——它只测了一个提交就宣布 `0af14f2 补充第 1 条说明` 是元凶，结论完全错。所以脚本崩了必须老老实实 `exit 125`。

以后再遇到"不知道哪次提交干的"这种烂摊子，我先别急着 `git log` 一格格翻，让它自己二分。
