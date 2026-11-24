---
title: '别下那 10 GB：几十 MB 就能看清一个量化模型里装了什么'
date: 2026-11-24 11:47:12
tags: [GGUF, 模型量化, HTTP Range, Hugging Face, 教程]
categories: [技术教程]
---

今天早上看到有人把一个 27B 模型塞进 10 GB 的文件里，说 3 bit 就能顶住 BF16。我第一反应是下下来看看，可 10 GB 得等好久。于是换了个思路：文件的头部其实写了它的一切，先用 HTTP Range 请求只取几十 MB，把头拆开数一遍。

### 第 0 步：先拿到文件清单和真实字节数

```bash
curl -sS "https://huggingface.co/api/models/nari-labs/Qwen3.8-27B-NariQ-GGUF/tree/main?recursive=1" \
  | python3 -c "import json,sys; [print(f\"{f['path']:42s} {f['size']:>13,}\") for f in json.load(sys.stdin) if f['path'].endswith('.gguf')]"
```

我这边输出：

```text
Qwen3.8-27B-NariQ-2.0bpw.gguf               6,753,954,304
Qwen3.8-27B-NariQ-2.5bpw.gguf               8,443,119,712
Qwen3.8-27B-NariQ-3.0bpw.gguf              10,123,626,592
Qwen3.8-27B-NariQ-mmproj-BF16.gguf           931,145,760
```

判定标准是字节数，不是"大概 10 GB"：3.0bpw 那个文件是 10,123,626,592 字节，按十进制算正好 10.12 GB，和 README 表格里写的 10.12 GB 对得上。

### 第 1 步：不下载，先把哈希对上

仓库里有 `SHA256SUMS`。我先想：能不能不下文件就知道它完整？HF 的 `resolve` 接口会回一个 `x-linked-etag`，那正是这个文件的 sha256：

```bash
BASE=https://huggingface.co/nari-labs/Qwen3.8-27B-NariQ-GGUF
curl -sSI -L "$BASE/resolve/main/Qwen3.8-27B-NariQ-3.0bpw.gguf" | grep -i '^x-linked-etag:'
grep '3.0bpw' "$BASE/raw/main/SHA256SUMS"
```

第一次跑，我四个文件全打印 MISMATCH，吓了一跳。原因是重定向有两段响应头，第一段里有一行 `access-control-expose-headers: ... X-Linked-ETag ...`，我原来的 `grep -i 'x-linked-etag'` 把这行也捞了进来。修法是让匹配锚到行首、并取最后一段：

```bash
etag=$(curl -sSI -L "$BASE/resolve/main/$f" | grep -i '^x-linked-etag:' | tail -1 | tr -d '\r"' | awk '{print $2}')
sha=$(awk -v n="$f" '$2 == n {print $1}' SHA256SUMS)
[ "$etag" = "$sha" ] && echo MATCH || echo MISMATCH
```

改完四个文件全是 MATCH。**假警报比真错误更费时间**，这就是第一个坑。

### 第 2 步：只取前 32 字节

```bash
curl -sS -L -r 0-31 -D - "https://huggingface.co/nari-labs/Qwen3.8-27B-NariQ-GGUF/resolve/main/Qwen3.8-27B-NariQ-3.0bpw.gguf" -o head.bin
```

响应里有两行关键信息：

```text
x-linked-size: 10123626592
content-range: bytes 0-31/10123626592
```

`content-range` 告诉你整个文件多大、这次只给了哪一段；`content-range` 末尾那个数字正好是第 0 步拿到的字节数，两处对得上。文件开头 4 字节是魔数 `GGUF`，后面 4 字节是版本号 `03 00 00 00`（GGUF v3），再往后 8 字节是张量个数 2710。

### 第 3 步：把整个头部读下来

只取 4 MB 会炸：我第一次按 4 MB 请求，解析器在 offset 4000006 处报 `struct.error: unpack_from requires a buffer of at least 4000014 bytes`。原因是元数据里塞了 556 个键值，其中 515 个是 `trellis.desc.*` 数组，还有整个 chat template，头部本身就有 11,146,720 字节。请求 40 MB 就够。

我写了个只依赖标准库的脚本 `~/scratch/gguf-head.py`：用 `curl -r 0-41943039` 取头部，然后按 GGUF 的格式顺序解析——magic、版本、张量数、KV 数、KV 表、张量表。它输出的关键几行：

```text
GGUF v3  tensors=2710  kv=556  arch=qwen35  ctx=262144
  type  46 trl3  tensors=  199 elems=12,924,354,560 bytes= 4,846,632,960
  type  47 trl4  tensors=   99 elems= 5,767,823,360 bytes= 2,883,911,680
  type  45 trl2  tensors=   82 elems= 7,474,380,800 bytes= 1,868,595,200
  type  48 trl5  tensors=  132 elems=   647,495,680 bytes=   404,684,800
  type  49 trl6  tensors=    1 elems=   79,298,560 bytes=   59,473,920
量化权重合计: 26,893,352,960 元素 / 10,063,298,560 字节 = 2.9935 bit/weight
```

顺手一个自己踩的坑：读版本号前必须跳过 4 字节魔数。我第一版忘了跳，版本打印成 1179993927 这种鬼数字，还以为文件坏了。

![一块被剖开的数据板：里面是发着金光的格子阵列，身后是一整面抽屉墙](/images/gguf-header-range-request.jpg)

### 第 4 步：算它到底压到了几 bit

44 到 49 这几个类型号不是上游 GGUF 的标准类型，是他们自己 fork 里新加的：fork 的 `ggml/src/ggml.c` 里写着 `TRL1` 到 `TRL6`，块大小 16 个元素、每块字节数是位数的两倍，所以 trl3 就是 3 bit/weight、trl5 就是 5 bit。我按这套规则把三个文件都算了一遍：

| 文件 | 文件字节 | 头部 | 张量数据 | 实测位宽 |
| --- | ---: | ---: | ---: | ---: |
| 3.0bpw | 10,123,626,592 | 11,146,720 | 10,112,479,872 | 2.9935 bit |
| 2.5bpw | 8,443,119,712 | 11,146,720 | 8,431,972,992 | 2.4936 bit |
| 2.0bpw | 6,753,954,304 | 11,151,872 | 6,742,802,432 | 1.9908 bit |

这里最值得说的是：**"3.0 bpw" 是个平均数，不是每个张量都 3 bit**。3.0 那个文件里 199 个张量是 trl3、99 个 trl4、82 个 trl2、132 个 trl5，连输出头都按频率切成 16 个带，`output.band00` 是 6 bit，`band02` 是 3 bit，其余大半是 2 bit。而 2.0bpw 那个文件里居然还有 12 个 1 bit 的张量。同一个模型，不同层给不同位宽——README 里的表格只给平均值，不看头部根本看不出这件事。

### 第 5 步：体积对账，确认我没读错

最硬的一条判定标准：头部字节加上所有张量字节，必须正好等于文件字节。

```text
3.0bpw  11,146,720 + 10,112,479,872 = 10,123,626,592  差 0 字节
2.5bpw  11,146,720 +  8,431,972,992 =  8,443,119,712  差 0 字节
2.0bpw  11,151,872 +  6,742,802,432 =  6,753,954,304  差 0 字节
```

连视觉投影头那个 931,145,760 字节的小文件也对得上（19,552 + 19,332,032 + 455,897,088×2 = 931,145,760，差 0）。差值不为 0，就说明我按 type_size 推算的大小规则错了，得回去重读 `ggml.c`。

### 第 6 步：还是真下了一个，验一遍

光靠头部推算，我总归不踏实，所以把最小的 2.0bpw 完整下下来跑了一遍 `sha256sum`：

```text
3a9e4955329dba8ba0be5509a9552220e5ccdae2f66bcc72c21c4580cd49bbb2  Qwen3.8-27B-NariQ-2.0bpw.gguf
```

和 `SHA256SUMS` 里那一行、`x-linked-etag` 三方一致。6,753,954,304 字节下了 3060 秒，均速 2.21 MB/s。

还有一件事得提醒：这些文件用了 fork 专有的类型号 44–49，上游 llama.cpp、Ollama、LM Studio 现在都装不下，得用他们那个 fork（MIT 协议，release `b12431-nariv0`）。所以在我这台没有 N 卡的小机器上，我只做到了"把文件拆开数清楚"，真跑起来还差着一块显卡。

这一个早上我最大的收获不是那个 2.99 bit，而是"别信文件名，先把头读完"。以后再看到谁说某个模型压到了几 bit，我大概还是会先问一句：那是平均值，还是每个张量都这样？