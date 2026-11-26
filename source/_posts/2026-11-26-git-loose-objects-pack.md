---
title: 'Git 仓库里散着 1531 个小盒子：松散对象、pack 和 git gc'
date: 2026-11-26 10:18:32
tags: [Git, 存储, 程序员工具, 教程]
categories: [技术教程]
---

今天整理仓库时，我盯着 `git count-objects` 的输出发了会儿呆：这个博客仓库里有 `1531` 个松散对象，占 `31.05 MiB`，已经打包的却只有 `22` 个。文件明明都提交过了，为什么还散着这么多小盒子？

Git 不只存“文件内容”。每次提交产生的文件快照、目录结构和提交记录，都会按内容哈希存成对象。对象刚写出来时，是躺在 `.git/objects/` 下的独立小文件；等 Git 把它们压成一个 pack，目录才安静下来。

## 先数清仓库里有什么

```bash
git count-objects -vH
```

这次输出是：

```text
count: 1531
size: 31.05 MiB
in-pack: 22
packs: 1
size-pack: 43.98 KiB
```

`count` 是松散对象数，`in-pack` 是已经装进 pack 的对象数，`packs` 是 pack 文件数量。判定标准是 `garbage: 0`；如果这里不是 0，先别急着 gc，可能有写坏的临时对象。

再看这些对象分别是什么类型：

```bash
git cat-file --batch-check='%(objecttype)' --batch-all-objects \
  | sort | uniq -c
```

输出为 `517 blob`、`210 commit`、`826 tree`，合计正好 `1553`，和上面的 `1531 + 22` 相等。blob 是文件内容，tree 记录目录与文件名，commit 指向一次提交和它的父提交；每个 blob、tree、commit 都是一个独立对象。

## 在临时仓库里看一次完整变化

为了不动正在写博客的仓库，我在临时目录创建了 20 次提交，每次新增一个小文件：

```bash
for i in $(seq 1 20); do
  printf 'demo line %s\n' "$i" > "file-$i.txt"
  git add "file-$i.txt"
  git commit -qm "add file $i"
done
git count-objects -vH
```

结果是 `count: 60`、`size: 240.00 KiB`、`in-pack: 0`。20 个文件为什么变成 60 个对象？因为除了 20 个 blob，每次提交还各留下 1 个 tree 和 1 个 commit，正好是 `20 × 3 = 60`。

此时对象分散在 `.git/objects/` 下。我用 `find` 数到 60 个文件，又跑了一遍完整性检查：

```bash
find .git/objects -mindepth 2 -maxdepth 2 -type f | wc -l
git fsck --full
```

第一条输出 `60`，第二条没有报错，退出码是 0；这表示对象内容与哈希能对上，可以进入整理阶段。

## 用 git gc 把它们装进 pack

```bash
git gc
git count-objects -vH
```

整理后输出变成：

```text
count: 0
size: 0 bytes
in-pack: 60
packs: 1
size-pack: 6.99 KiB
```

松散对象从 60 变成 0，同一批对象全部进入了 1 个 pack，体积从 240 KiB 收到 6.99 KiB。pack 旁会出现 `.pack`、`.idx` 和 `.rev` 三个文件；`.idx` 是查对象的索引，`.rev` 用来辅助反向遍历。

我没有只看“文件少了”就收工，而是让 Git 重新验一遍：

```bash
idx=$(find .git/objects/pack -name '*.idx' -print -quit)
git verify-pack -v "$idx" | grep -Ec '^[0-9a-f]{40} '
git fsck --full
```

第一条输出 `60`，表示 pack 的索引里能核对到 60 个对象；第二条仍然没有报错，退出码为 0。判定标准是对象数前后相等、pack 能验证、`fsck` 通过，三项都对，才算整理成功。

如果 `git gc` 后 `fsck` 报 `missing` 或 `bad`，先别反复 gc；用 `git fsck --full` 记住缺失对象的哈希，再从另一份克隆或远端取回对应对象。现在我看见这 1531 个小盒子，就知道它们不是垃圾，只是还没被 Git 收进那个安静的 pack。