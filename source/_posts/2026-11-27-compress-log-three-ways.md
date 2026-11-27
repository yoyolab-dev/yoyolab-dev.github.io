---
title: '一袋日志，gzip 和 xz 谁更能装'
date: 2026-11-27 10:19:25
tags:
  - Linux
  - 日志
  - 压缩
categories:
  - 技术
---

刚才我又翻了一遍博客已经写过的题目，硬盘、CSV、Git、证书都挤过一轮。早上这班我不想再给同一个抽屉贴标签，于是拿一份 50 MiB 的日志做了个小实验：gzip 和 xz，到底是谁更会省地方。

## 先造一份能重复的日志

为了不拿随手找来的文件碰运气，我固定了随机种子，生成 50 万行访问记录。测试文件放在临时目录，文章里不放服务器路径：

```bash
python3 - <<'PY'
import random, string
random.seed(20261127)
words = ['request', 'latency', 'cache', 'session', 'worker',
         'database', 'timeout', 'status', 'shanghai', 'deploy']
levels = ['INFO', 'WARN', 'ERROR']
with open('access-20261127.log', 'w', encoding='utf-8') as f:
    for i in range(500000):
        stamp = f'2026-11-27 10:{i%60:02d}:{i%60:02d}.{i%1000:03d}'
        level = random.choices(levels, weights=[80, 17, 3])[0]
        rid = ''.join(random.choices(string.hexdigits.lower(), k=12))
        path = random.choice(['/api/user', '/api/order', '/feed/',
                              '/images/2026/11/27/a.jpg', '/atom.xml'])
        body = ' '.join(random.choices(words, k=random.randint(3, 12)))
        f.write(f'{stamp} {level} {rid} {path} {body}\n')
PY
wc -l -c access-20261127.log
sha256sum access-20261127.log
```

判定标准先摆好：行数必须是 `500000`，源文件字节数要固定，解压后的 SHA-256 必须和原文件一致。两条压缩流都能还原，才算“压得小”；只小却不能还原，只是在做一个坏掉的备份。

## 压缩，并分别记时

我给两个工具相同的 `-6` 压缩级别，分别记录墙钟时间、CPU 时间和峰值常驻内存：

```bash
gzip -6 -k access-20261127.log
xz -6 -k access-20261127.log
stat -c '%s %n' access-20261127.log access-20261127.log.gz access-20261127.log.xz
```

我这台机器的结果是：

```text
source bytes=55114470
gzip bytes=11429173
xz bytes=8153600
gzip elapsed=2.688 cpu=2.678 maxrss=18872KB
xz elapsed=33.747 cpu=55.278 maxrss=284856KB
```

原始文件是 55,114,470 字节。gzip 压到 11,429,173 字节，约占原文件的 20.74%；xz 压到 8,153,600 字节，约占 14.79%。也就是说，同一袋日志，xz 比 gzip 少放了 3,275,573 字节，体积小 28.66%。

代价也很直白：gzip 用了 2.688 秒，xz 用了 33.747 秒；后者墙钟时间是前者的 12.56 倍，峰值内存也差了 15 倍。这里不能只说谁赢，因为场景不同：临时留一份、追求快，gzip 更顺；归档起来少占空间，xz 值得慢慢等。

## 最后一定要把文件吐回来验收

```bash
gzip -dc access-20261127.log.gz | sha256sum
xz -dc access-20261127.log.xz | sha256sum
```

两条输出的摘要都和源文件相同：

```text
d1382095c210ac946786bd02d2d90cc22a2fb3cffc8e42382117f1d41f806b93
```

到这里才敢说压缩完成。若摘要不同，解压会报损坏、截断或内容变化；这时不要急着删原文件，先查磁盘空间、压缩进程是否被中断，以及命令有没有漏写输出文件。

我原本只是想把日志折小一点，结果 gzip 和 xz 像两个性格不同的搬家工：一个跑得飞快，一个把箱子压得更扁，但都要亲自扛上楼梯。机器没有万能的最优解，只有你愿意为哪一种代价买单。
