---
title: '服务器没装 jq，我用 Python 把 JSON 拆开看'
date: 2026-11-20 10:18:18
tags: [Python, JSON, 命令行, 程序员工具]
categories: [技术教程]
---

今天我又撞上了那个有点熟悉的场景：脚本要读一份 JSON，可服务器上没有 `jq`。以前我会先找一句最短的 `grep`，能跑就算完；这次我不想再赌字段名和引号，所以老老实实用了 Python 自带的 `json`。它不漂亮，但每一步都能看清。

先准备一份很小的样例，免得教程只剩概念：

```bash
mkdir -p json-read-demo
cd json-read-demo
printf '%s\n' '{"repo":"yoyolab-dev.github.io","stars":42,"open_issues":3,"owner":{"name":"yoyolab-dev"},"topics":["blog","hexo"]}' > sample.json
```

### 1. 先确认工具在不在

```bash
python3 --version
command -v jq
```

我这里第一行输出 `Python 3.14.9`，第二行没有任何路径，退出码是 `1`，说明 `jq` 确实没装。判定标准很简单：`command -v` 能打印路径才算有；空输出不是“命令安静”，就是不存在。

### 2. 先把 JSON 摊开给人看

```bash
python3 -m json.tool sample.json
```

真实输出是：

```text
{
    "repo": "yoyolab-dev.github.io",
    "stars": 42,
    "open_issues": 3,
    "owner": {
        "name": "yoyolab-dev"
    },
    "topics": [
        "blog",
        "hexo"
    ]
}
```

如果这里报 `JSONDecodeError: Expecting value`，多半是末尾多了一个逗号，或者字符串里的双引号没转义。报错里的 `line 1 column 25` 会指出出错的行列，先修那一处，不要整份重写。

### 3. 再取真正需要的字段

新建 `pick.py`：

```python
import json

with open("sample.json", encoding="utf-8") as f:
    d = json.load(f)

print(f"{d['repo']}: {d['stars']} stars")
print("owner:", d["owner"]["name"])
print("topics:", ", ".join(d["topics"]))
```

运行 `python3 pick.py`，我这里得到三行：

```text
yoyolab-dev.github.io: 42 stars
owner: yoyolab-dev
topics: blog, hexo
```

这里有个细节我很喜欢：`json.load()` 会把整个文件交给解析器，不是按行猜。少逗号、括号没配对、字符串没闭合，都会在读取时直接报错，不会悄悄把后半张 JSON 当成另一段东西。

### 4. 字段不存在时，别让脚本突然摔倒

如果我误写了 `d["language"]`，输出是：

```text
Traceback (most recent call last):
  File "<stdin>", line 3, in <module>
KeyError: 'language'
```

退出码是 `1`。这其实很诚实：这个字段本来就不存在。偶尔我只是想顺手看看有没有语言信息，就会改用：

```python
print("language:", d.get("language", "not provided"))
print("has language:", "language" in d)
```

对应输出：

```text
language: not provided
has language: False
```

`get()` 适合“有就用，没有也无妨”的字段；要确认字段是否真的存在，就用 `in`。这比一串脆弱的 `grep` 多写两行，却少了一种最难查的错：明明抓到了半截字符串，还以为程序读对啦。

我现在更喜欢先把 JSON 打印出来看一眼，再决定究竟要哪几个键。工具少一点没关系，至少每一步的结果都看得见。
