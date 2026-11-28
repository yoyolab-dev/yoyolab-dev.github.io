---
title: '让 make 同时干八件事：-j 的加速和一次翻车'
date: 2026-11-28 15:19:06
tags:
  - Make
  - Linux
  - 并行构建
categories:
  - 技术
---

今天想把八件互不相干的事一次干完。Makefile 默认一次只干一件，于是我先把八件各睡一秒的活排成队，再把队伍拆开跑。拆之前我想先问清楚：到底快了多少，又会在哪儿翻车。

![我跑 make -j 时终端里的实测输出](/images/2026-11-28-make-parallel-jobs.jpg)

## 第一步：八个活，先老老实实排队

先造一个最简单的 Makefile，八个目标各睡一秒，谁也不认识谁：

```make
JOBS := 8
TARGETS := $(addprefix out,$(addsuffix .txt,$(shell seq 1 $(JOBS))))
all: $(TARGETS)
$(TARGETS):
	@echo "building $@"
	@sleep 1
	@echo done > $@
```

注意配方那几行前面是**真 Tab**，写成空格 make 会直接报 `missing separator`。我的机器是 GNU Make 4.4.1（`make --version` 能看到），计时用 `export TIMEFORMAT='%3R'`，结果：

```text
$ time make -s            elapsed 8.046 s
$ time make -s -j8        elapsed 1.095 s
$ time make -s -j4        elapsed 2.013 s
$ time make -s -j16       elapsed 1.012 s
$ time make -s -j         elapsed 1.010 s
```

判定标准很简单：`ls out*.txt | wc -l` 得 8，每个文件内容是 `done`。八倍并行比串行快了七秒多；而 `-j` 不带数字是不限上限的，有几个能同时跑就几个——在共享 CI 上这么写，很容易把别人的活一起挤死，所以我习惯写 `-j$(nproc)`。

## 第二步：换成真吃 CPU 的活，天花板露出来了

上面那八个任务只是睡觉，不抢 CPU。真换成八个 python 计算任务（单个 0.84 秒），数字就不好看了：

```text
$ time make -s -f cpu.mk        elapsed 7.470 s
$ time make -s -f cpu.mk -j4    elapsed 4.086 s
$ time make -s -f cpu.mk -j8    elapsed 4.064 s
$ time make -s -f cpu.mk -j2    elapsed 4.334 s
```

`-j4` 只快了 1.8 倍，不是 4 倍。原因不在 make：`nproc` 报 4 核，可 `cat /sys/fs/cgroup/cpu.max` 打出来是 `200000 100000`——每 100 毫秒只给 200 毫秒 CPU，实际配额就 2 核。我又用 `seq 1 4 | xargs -P 4` 复测：1 个任务 0.958 秒，2 个并发 1.127 秒，4 个并发 2.215 秒，真并行度确实是 2。容器里 `nproc` 会骗人，想知道能并行多少，先看 `cpu.max`。

## 第三步：一开 -j 就翻车

并行不是白捡的。我造了个坑：`split` 这条规则会生成 `part-aa` 和 `part-ab` 两个文件，可 Makefile 只声明了 `split` 一个目标，而 `report` 又自己去读 `part-aa`。

```make
all: split report
split: input.txt
	@sleep 0.5
	@split input.txt part-
report: input.txt
	@wc -l < part-aa > report
```

先 `seq 1 100 > input.txt` 造输入。串行时 make 按顺序先跑 split，report 拿到文件，退出码 0，一切安好。可一加 `-j8`，两个目标同时开跑，report 抢先一步：

```text
/bin/sh: 1: cannot open part-aa: No such file
make: *** [race.mk:8: report] Error 2
make: *** Waiting for unfinished jobs....
```

退出码 2。修法有两种，我都试了：把依赖写实，`report: split`；或者用 GNU Make 4.3 之后才有的分组目标写法 `part-aa part-ab &: input.txt`，让 make 知道这几个文件是一起产出的，report 依赖 `part-aa` 就会老实等。两种改法在 `-j8` 下都退出码 0。

## 第四步：并行以后，输出会自己串台

日志混在一起也是并行附赠的赠品。四个目标各打三行，`make -j4` 的输出是 `s1 line1 / s2 line1 / s3 line1 / s4 line1` 这样交叉着来的；加一个 `--output-sync=target`，同一个目标的输出就被攒到一起了：

```text
$ make -f sync.mk -j4 --output-sync=target
s1.txt line1
s1.txt line2
s1.txt line3
s2.txt line1
...
```

CI 上日志被搅乱，出了错都难定位，这个开关值得加。

我喜欢 `-j8` 那 1.095 秒，比串行的 8.046 秒短了整整七秒。但更喜欢翻车那次——串行时安静通过的东西，一开并行就炸，这种坑不亲手跑一遍真的记不住。
