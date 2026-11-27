---
title: '文件开头藏着三个字节：UTF-8 BOM 到底该不该留'
date: 2026-11-27 19:19:10
tags: [UTF-8, BOM, JSON]
categories: [技术]
---

下午我又撞见一个很像玄学的报错：Node 的 `JSON.parse` 说文件开头多了一个不认识的字符，可编辑器里明明第一眼就是 `{`。这次我没猜，先把文件最前面的几个字节掏出来看了看。

## 先造两份只差三个字节的文件

在工作目录里运行：

```bash
mkdir -p bom-demo
cd bom-demo
python3 -c "from pathlib import Path; Path('plain.json').write_bytes(b'{\"name\":\"mimiya\"}\n'); Path('bom.json').write_bytes(b'\xef\xbb\xbf{\"name\":\"mimiya\"}\n')"
```

`plain.json` 是普通 UTF-8，`bom.json` 只在前面多塞了 `EF BB BF`。运行后应生成两个 18 字节左右的小文件；判定标准是肉眼看不出区别。

## 用 od 看真正的开头

```bash
od -An -tx1 -N16 plain.json
od -An -tx1 -N16 bom.json
```

我这台机器的实际输出是：

```text
 7b 22 6e 61 6d 65 22 3a 22 6d 69 6d 69 79 61 22
 ef bb bf 7b 22 6e 61 6d 65 22 3a 22 6d 69 6d 69
```

第一行以 `7b` 开头，那是 `{`；第二行先出现 `ef bb bf`，随后才是同样的 `7b`。这三个字节就是 UTF-8 BOM——编辑器通常把它藏起来，但解析器看得见。

## 再让程序亲口报错

用 Node v26.12.1 读取 `bom.json`：

```bash
node -e "const fs=require('fs'); console.log(JSON.parse(fs.readFileSync(process.argv[1],'utf8')))" bom.json
```

报错里的箭头正指向那个看不见的字符：

```text
SyntaxError: Unexpected token '﻿', "﻿{\"name\":\"mimiya\"}\n" is not valid JSON
```

Python 3.14.9 的表现还更细：`json.loads()` 直接吃字节时能识别 BOM，但先按普通 UTF-8 解码成文本就会失败。

```bash
python3 -c "import json,pathlib; print(json.loads(pathlib.Path('bom.json').read_text(encoding='utf-8')))"
```

```text
json.decoder.JSONDecodeError: Unexpected UTF-8 BOM (decode using utf-8-sig): line 1 column 1 (char 0)
```

所以“Python 能读”也得看它读的是 bytes 还是 str，不能一概而论。

## 修的时候先确认，别盲删

Node 可以在解析前只去掉开头的 BOM：

```bash
node -e "const fs=require('fs'); const s=fs.readFileSync(process.argv[1],'utf8').replace(/^\uFEFF/, ''); console.log(JSON.parse(s))" bom.json
```

通过标准是输出 `{'name': 'mimiya'}`，不再抛异常。Python 文本读取则把编码改成 `utf-8-sig`，它会在解码时吃掉 BOM。

如果要从源头清掉文件，先检查前三个字节，再有条件地截掉：

```bash
python3 -c "from pathlib import Path; p=Path('bom.json'); b=p.read_bytes(); p.write_bytes(b[3:] if b.startswith(bytes.fromhex('efbbbf')) else b)"
```

修完再跑一次 `od`，开头应当变成 `7b 22 6e 61`。这里不能直接对所有文件执行“删前三个字节”，否则没有 BOM 的文件会平白丢掉 `{`。

BOM 不是乱码，也不是加密，它只是文件开头的一小段标记。UTF-16 经常靠它判断编码，Excel 也可能因此更愿意认出 UTF-8；可对普通 UTF-8 和 JSON 来说，它通常没有必要，还会让不同解析器表现不一。我现在再看到 `Unexpected token`，会先去看文件最前面的字节，而不是先怀疑引号和逗号。
