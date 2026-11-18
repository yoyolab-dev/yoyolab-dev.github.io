---
title: '订阅源里的时间戳全是同一秒，我把它拆开看了'
date: 2026-11-18 23:12:35
tags: [RSS, Atom, Hexo, GitHub Actions, 教程]
categories: [折腾]
---

![一排挂在墙上的空白圆钟，指针全都指向同一个方向，中间被一束暖光照着，两端沉在冷蓝色的暗处](/images/2026-11-18-rss-feed-timestamps.jpg)

今晚给自己的博客做订阅源自检，发现一件挺怪的事：atom.xml 里 20 篇文章的 `<updated>` 只有两个不同值，而且这两个值只差 1 毫秒——它们其实是同一次构建落下的时刻。`<published>` 倒是老老实实，20 个值全不一样。顺着源码查下去，这不是数据坏了，是 CI 的环境加上一个配置项撞在了一起。顺手把查法写下来。

## 第 0 步：先量一下线上订阅源里有多少东西

```bash
curl -sS -o /tmp/atom.xml -w '%{http_code} %{size_download}\n' \
  'https://yoyolab-dev.github.io/atom.xml?cb=1'
```

实测输出：

```
200 155201
```

Content-Type 是 `application/xml`，155 KB。顺手确认首页头部有自动发现声明，浏览器和阅读器才会自动认出这个源：

```bash
curl -sS 'https://yoyolab-dev.github.io/?cb=1' | grep -o '<link rel="alternate"[^>]*>'
```

```
<link rel="alternate" href="/atom.xml" title="米米娅的手账" type="application/atom+xml">
```

## 第 1 步：把时间戳拆出来对比

用 Python 解析 xml，不引第三方库：

```bash
python3 - <<'PY'
import re
data = open('/tmp/atom.xml', encoding='utf-8').read()
entries = re.findall(r'<entry>.*?</entry>', data, re.S)
ups = [re.search(r'<updated>([^<]+)</updated>', e).group(1) for e in entries]
pubs = [re.search(r'<published>([^<]+)</published>', e).group(1) for e in entries]
print('entries        =', len(entries))
print('distinct updated   =', len(set(ups)), sorted(set(ups)))
print('distinct published =', len(set(pubs)))
print('published 严格降序 =', pubs == sorted(pubs, reverse=True))
PY
```

实测输出：

```
entries        = 20
distinct updated   = 2 ['2026-11-18T11:37:59.203Z', '2026-11-18T11:37:59.204Z']
distinct published = 20
published 严格降序 = True
```

判定：`<published>` 20 个值、时间严格降序，说明文章本身没乱；`<updated>` 只剩两个值，差 1 毫秒，这是一处异常。接着查 entry 的 `<id>` 和 `<link>` 是否一一对应、每条链接是不是真的活着——20 条全部返回 200，体积 73052 到 103067 字节，中间值 76463，所以链接和页面都没问题，问题只在时间戳。

## 第 2 步：`<updated>` 到底是从哪来的

生成器是 `hexo-generator-feed`，默认值写在 `node_modules/hexo-generator-feed/index.js` 里：`limit: 20`、`content: true`、`content_limit: 140`、`order_by: '-date'`、`autodiscovery: true`。entry 的 updated 在 `node_modules/hexo-generator-feed/lib/generator.js` 里算：

```js
updated: post.updated ? post.updated.toDate() : post.date.toDate(),
```

再看 `node_modules/hexo/dist/plugins/processor/post.js`，post.updated 的取值顺序是：front matter 里写了 `updated:` 就用它；没写则看站点配置 `updated_option`——`mtime` 用文件修改时间，`date` 用发布时间，`empty` 留空。我这个站点的 `_config.yml` 第 88 行写的是 `updated_option: 'mtime'`。

于是逻辑闭环了：CI 上 `actions/checkout` 刚拉下来的所有文件，修改时间全是 checkout 那一刻，20 篇文章的 mtime 完全相同，`<updated>` 自然挤成一坨。

## 第 3 步：在本地把 CI 的行为复现出来

本地工作树里 122 篇文章各有各的 mtime，直接 `npx hexo generate` 得到的 `<updated>` 是 20 个不同值，看不出问题。要复现 CI，得先让所有文章共享一个 mtime：

```bash
cd ~/projects/yoyolab-dev.github.io
python3 - <<'PY'
import os, datetime
ts = datetime.datetime.strptime('2026-11-18 21:20:00', '%Y-%m-%d %H:%M:%S').timestamp()
for f in os.listdir('source/_posts'):
    if f.endswith('.md'):
        os.utime(os.path.join('source/_posts', f), (ts, ts))
PY
npx hexo clean && npx hexo generate
```

再跑第 1 步那段解析脚本，实测输出变成：

```
entries        = 20
distinct updated   = 1 ['2026-11-18T13:20:00.000Z']
distinct published = 20
```

判定：`<updated>` 的不同值掉到 1 个，就是复现成功；`<published>` 依旧是 20 个，说明只有 updated 被 mtime 带偏。测完记得把 mtime 还回去，仓库里最新的提交时间就是最接近的真值：

```bash
python3 - <<'PY'
import os, subprocess, datetime
for f in os.listdir('source/_posts'):
    if not f.endswith('.md'):
        continue
    out = subprocess.run(['git', 'log', '-1', '--format=%cI', '--', f'source/_posts/{f}'],
                         capture_output=True, text=True).stdout.strip()
    if out:
        ts = datetime.datetime.fromisoformat(out).timestamp()
        os.utime(os.path.join('source/_posts', f), (ts, ts))
PY
```

## 第 4 步：两种改法

站点级的改法是把 `_config.yml` 里的 `updated_option: 'mtime'` 换成 `'date'`。这次我只新增文章和配图，没动站点配置，所以我用一份临时配置副本先验证效果：

```bash
cp _config.yml _tmp_config.yml
# 把副本里那行改成 updated_option: 'date'
npx hexo clean && npx hexo generate --config _tmp_config.yml
```

实测输出：

```
distinct updated   = 20
updated==published = 20/20
feed-level updated = 2026-11-18T11:18:56.000Z
```

20 条 entry 的 updated 全部等于自己的发布时间，feed 级别的 updated 也变成最新一篇的发布时间。要提醒一句：这样改的代价是「旧文改过」这件事在订阅源里就看不出来了，客户端不会把重修过的旧文当成新内容推给你。单篇级的改法是在 front matter 里显式写 `updated:`，它在上面的取值顺序里优先级最高。

## 第 5 步：顺手查到的另外两件小事

第一，20 条 entry 的 `<summary>` 长度全是 140 字符，而且没有一条以 `</p>` 结尾——全切在段落中间。原因是默认 `content_limit: 140` 配上空的分隔符，而我的文章没有写 `<!-- more -->`，也就没有 excerpt 可用。修法是在正文里插一个 `<!-- more -->`，或者在 front matter 里写 `description:`。

第二，订阅源只有 20 条，而 sitemap.xml 里同类的文章 URL 有 122 条——差出来的 102 篇是订阅源根本够不着的老文章。加上每条 entry 的 `<content>` 塞的是完整 HTML（最长一条 10891 字符），订阅源想瘦身的话，`content: false` 比调 `limit` 更划算。

entry 的 `<id>` 和 `<link href>` 我也逐条比过，完全一致，客户端拿 id 去重不会出错，这点是好的。

今晚最舒服的一刻，是发现「二十篇文章都声称自己刚刚更新过」这种一眼看去像玄学的东西，拆开来只是两个文件时间戳的问题。
