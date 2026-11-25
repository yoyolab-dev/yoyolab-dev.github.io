---
title: '锟斤拷不是病毒，是编码打了个结'
date: 2026-11-25 15:18:36
tags: [编码, iconv, 乱码, 排查, 教程]
categories: [技术教程]
---

下午我打开一份 CSV，第一行就躺着一句“锟斤拷”。第一反应是文件坏了——删掉重下吧？先别急。那三个字不是病毒，也不是文件里原装的字，它是一个字符被按错的编码读了两遍之后留下的痕迹。

![打成一团的铜线缠着一条浅色丝带，线头接着一枚琥珀色小灯泡，静静躺在深色木桌上](/images/2026-11-25-mojibake-encoding-chain.jpg)

下午剩下的时间我就在临时目录里把这事从字节层面捋了一遍，下面每条命令和输出都是这台机器上实跑的。

### 第一步：造一份中文，再存两版

```bash
mkdir -p ~/encoding-lab && cd ~/encoding-lab
printf '米米娅的手账\n上海\n' > demo.txt
wc -c demo.txt
od -An -tx1 demo.txt
```

`wc -c` 给出 26，字节摊开是这样：

```
 e7 b1 b3 e7 b1 b3 e5 a8 85 e7 9a 84 e6 89 8b e8 b4 a6 0a
 e4 b8 8a e6 b5 b7 0a
```

“米”在 UTF-8 里是 `e7 b1 b3`，一个汉字占三格。这台机器上没装 `file` 和 `xxd`，`od` 是 coreutils 自带的那个，`-An` 不打地址，`-tx1` 按十六进制一字节一组列出来。

再用 iconv 把它转成 GBK：

```bash
iconv -f UTF-8 -t GBK demo.txt > demo-gbk.txt
wc -c demo.txt demo-gbk.txt
od -An -tx1 demo-gbk.txt
```

输出 `26 demo.txt`、`18 demo-gbk.txt`。GBK 里“米”变成了 `c3 d7`，两个字；18 = 8 个汉字 × 2 字节 + 两个换行。两份文件内容一模一样，差的只是每个字占了几个字节。

### 第二步：用错编码读，第一层乱码长这样

```bash
python3 -c "print(open('demo.txt', encoding='gbk').read())"
```

屏幕上不是“米米娅的手账”，而是：

```
绫崇背濞呯殑鎵嬭处
涓婃捣
```

文件一个字节都没坏，只是被人用错了尺子去量：UTF-8 的字节被当成 GBK 两两切开，每个汉字被拆成两个别的字。

### 第三步：锟斤拷是怎么长出来的

再往里错一层。我另存了一个带右单引号的版本（`’` 的 UTF-8 字节是 `e2 80 99`），按 GBK 读，并且允许替换：

```bash
python3 -c "raw=open('demo2.txt','rb').read(); s=raw.decode('gbk', errors='replace'); print(s); print('替换字符个数:', s.count(chr(0xfffd)))"
```

输出是 `绫崇背濞呪�檚 鎵嬭处`，这串 16 个字符（含两个换行）里混进了 1 个 `�`——那是 U+FFFD 替换字符，意思是“这里有个我读不懂的字节”。关键在它自己的 UTF-8 字节：`ef bf bd`。这三个字节要是再被当成 GBK 读，会被切成 `ef bf`（锟）、`bd ef`（斤）、`bf bd`（拷）。所以两个替换字符连着来，就正好读成“锟斤拷”：

```bash
python3 -c "print(b'\xef\xbf\xbd\xef\xbf\xbd'.decode('gbk'))"
```

屏幕上就是 `锟斤拷`。所以那三个字是“读错一次 → 换成 � → 再读错一次”的产物。

### 第四步：先猜它是哪一版编码

```bash
python3 -c "
raw=open('demo-gbk.txt','rb').read()
for enc in ('utf-8','gbk'):
    try:
        raw.decode(enc); print(enc,'OK')
    except UnicodeDecodeError as e:
        print(enc,'FAIL',e.reason)
"
```

对 GBK 那份，输出是 `utf-8 FAIL(invalid continuation byte)`、`gbk OK`。判定标准就一条：utf-8 报 invalid continuation byte、gbk 全过，那基本就是 GBK 文件。

反过来那个坑更要命：纯中文的 UTF-8 文件，两种编码都能解出来（我这份 demo.txt 就是 `utf-8:OK gbk:OK`）。“能解开”不等于“解对了”，还得看解出来的字是不是人话。顺带一提，`iconv -l` 在这台机器上列了 1180 种编码，中文文件现实中基本就 UTF-8 和 GBK/GB18030 两派，先从这两派试起。

### 第五步：修，iconv 一行

```bash
cp demo-gbk.txt demo-gbk.txt.bak
iconv -f GBK -t UTF-8 demo-gbk.txt > fixed.txt
cmp fixed.txt demo.txt && echo 'cmp: identical'
md5sum demo.txt fixed.txt
```

输出 `cmp: identical`，两个 md5 一模一样：`44f58b5b0bc523575dfefe2a3f8f22f4`，字节数 18 → 26。再反向转回 GBK 和原件比，也是 identical。顺手检查修好的文件里有没有残留的替换字符：

```bash
python3 -c "print(open('fixed.txt',encoding='utf-8').read().count(chr(0xfffd)))"
```

输出 0，干净的。同一个文件用 UTF-8 去硬读原始 GBK 版本，替换字符个数是 11；读对了就是 0。

### 两个容易踩空的坑

一、别拿编辑器的“另存为 UTF-8”当修复。那只是把已经读错的内容重新编码存一遍，字符都错了，再存几次只会错得更深。先在字节层面用 iconv 转回来。

二、文件被截断时 iconv 会直接罢工。我把 demo.txt 砍到 23 字节，正好切在“海”字中间：

```bash
head -c 23 demo.txt > cut.txt
iconv -f UTF-8 -t UTF-8 cut.txt > /dev/null
```

输出 `iconv: incomplete character or shift sequence at end of buffer`，退出码 1。Python 说得更细：`UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe6 in position 22: unexpected end of data`。加 `-c` 能让它跳过坏字节继续（输出 24 字节，坏尾巴被丢掉），但退出码还是 1——它只是不拦你，没有替你把文件补回来。判定标准：退出码 1 而且位置指向文件末尾，多半是下载或拷贝被截了，重新取一份比修字节靠谱。

### 一张表收尾

| 现象 | 原因 | 修法 |
| --- | --- | --- |
| 满屏“锟斤拷” | 内容已被替换字符污染过，再按 GBK 读 | 找最早那份原件，或从来源重新下载 |
| “绫崇背濞呯殑”这类汉字乱码 | 编码猜错，UTF-8 被当成 GBK | `iconv -f GBK -t UTF-8 原文件 > 新文件` |
| iconv 报 invalid continuation byte | 源编码选错 | 换成 `-f UTF-8` 或 `-f GB18030` 再试 |
| iconv 报 incomplete character at end of buffer | 文件被截断 | 重新下载，别硬转 |
| 转完还是乱码 | 原文已被替换字符污染 | 查 `chr(0xfffd)` 的个数，不是 0 就说明存过备份 |

乱码这东西看着吓人，其实特别讲道理：每个字节都老老实实待在该待的位置，只是被人用错了尺子去量。以后再看到“锟斤拷”，我大概会先笑一下，再去翻那个文件当初是从哪台机器上拷来的——多半是某台 Windows 和某台 Linux 之间的一次握手失败。
