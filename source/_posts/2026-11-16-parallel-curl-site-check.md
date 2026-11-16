---
title: '把站点体检从四分钟压到十三秒：curl -Z 的并行实测'
date: 2026-11-16 15:26:19
tags: [curl, 命令行, 性能, 教程, 折腾]
categories: 折腾
---

![左边一条弯曲的细管里，琥珀色的小方块排成一列慢慢往前走；右边二十条直管并行，方块同时流动](/images/2026-11-16-parallel-curl-site-check.jpg)

周一中午想把攒了一周的全站链接体检跑完，等它的工夫我去泡了两趟茶。这活很简单：把站上所有 URL 挨个 curl 一遍看状态码，一共 449 条，串行跑下来快四分钟。今天决定把它一次解决掉——下面是同一个任务从 58 秒量到 13 秒的全过程，命令和我这边真实的输出都贴在下面，你照着敲能跑出一模一样的形状。

## 第 0 步：先量出来，再动手

```bash
curl --version | head -1
# curl 8.14.1 (x86_64-pc-linux-gnu) libcurl/8.14.1 OpenSSL/3.5.7 zlib/1.3.1 nghttp2/1.64.0

curl -sS 'https://yoyolab-dev.github.io/sitemap.xml' -o /tmp/sm-1116.xml \
  -w 'sitemap %{http_code} %{size_download} bytes\n'
# sitemap 200 89305 bytes

grep -c '<loc>' /tmp/sm-1116.xml
# 449
```

判定标准：状态码 200，`<loc>` 条数是 449。顺手看了眼这 449 条的构成：310 条标签页、120 条 `/2026/` 下的页面、16 条分类页，剩下首页和栏目页——sitemap 里一条图片 URL 都没有，图得另外抽，那是另一个脚本的事。

样本固定抽 60 条，带固定种子，保证后面三种跑法打的是同一批 URL：

```bash
python3 - <<'PY'
import random, re
urls = re.findall(r'<loc>(.*?)</loc>', open('/tmp/sm-1116.xml').read())
random.seed(1116)
with open('urls60.txt', 'w') as f:
    for i, u in enumerate(random.sample(urls, 60)):
        f.write(f'{u}?cb=1116i{i}\n')
PY
```

（尾巴上挂 `?cb=` 是绕 CDN 旧缓存，保证这条请求真的回源。60 条里 14 条是文章页，其余是标签页和归档页。）

## 第 1 步：串行基线

```bash
while read -r u; do
  curl -sS --max-time 25 -o /dev/null -w "%{http_code}\n" "$u"
done < urls60.txt
# 串行 while-read: 58.06s  200=60/60
```

58.06 秒，单条均摊 0.968 秒。判定标准：`200=60/60`。有非 200 就先把坏链修掉再谈提速——拿一条本来就 404 的链接当基准，量出来的都是噪声。

## 第 2 步：一条请求的时间都花在哪

```bash
curl -sS -o /dev/null \
  -w '%{time_namelookup} %{time_connect} %{time_appconnect} %{time_starttransfer} %{time_total}\n' \
  "https://yoyolab-dev.github.io/2026/11/15/2026-11-15-drawtext-colon-escape/?cb=warm1116"
# 0.006670 0.006822 0.437523 0.646208 0.854373
```

读法：DNS 6.7 ms、TCP 连接 6.8 ms、TLS 握手 437.5 ms、首字节 646.2 ms、总计 854.4 ms。DNS 和连接加起来还不到 14 ms，TLS 握手一个人吃掉一半多——而第 1 步那 60 条，每一条都把手重握了一遍。这就是要找的钱。

## 第 3 步：换成一个进程串着拉，省掉重复握手

不用并行也能砍掉一半时间：一个 curl 进程里按顺序给 60 条 URL，它会复用同一条连接。

```bash
curl -sS --max-time 25 -w '%{http_code} %{num_connects}\n' \
  -o /dev/null "URL1?cb=1116i0" -o /dev/null "URL2?cb=1116i1" ...   # 逐条给 -o
# 单进程串行 60 条: 30.60s  200=60/60  新建连接合计=1
```

30.60 秒，而且 60 条只新建了 1 条连接。省下来的 0.458 秒/条，正好等于第 2 步里那 437 ms 的握手。冷热对照更直白，同一个 URL 连着来两次：

```bash
curl -sS -o /dev/null -w '第1次 %{time_appconnect}s %{time_total}s conns=%{num_connects}\n' \
     -o /dev/null -w '第2次 %{time_appconnect}s %{time_total}s conns=%{num_connects}\n' "$U" "$U"
# 第1次 0.443182s 1.174105s conns=1
# 第2次 0.000000s 0.405187s conns=0
```

第二条的握手时间是 0，总时间从 1.17 秒掉到 0.41 秒。

这里踩了个坑要记一下：多 URL 时 `-o` 只管紧跟着它的那一条，后面的响应体会原样打到 stdout。我第一版脚本因此把整页 HTML 灌进了解析，崩在 `ValueError: invalid literal for int() with base 10: 'html>'`——看到那行才反应过来输出里混了 body。多 URL 就逐条给 `-o`，或者干脆走 `-K` 配置文件。

## 第 4 步：真要并行，用 curl 自带的 -Z

配置文件一行一条 URL，每条自己带 `output`，这样 write-out 还能留在 stdout 上：

```bash
grep -o '<loc>[^<]*' /tmp/sm-1116.xml | cut -c6- > urls.txt          # 449 条
while read -r u; do
  printf 'url = "%s?cb=full1116"\noutput = "/dev/null"\n' "$u"
done < urls.txt > c.conf

time curl -sS --max-time 25 -Z --parallel-max 10 -K c.conf \
  -w '%{http_code} %{url_effective}\n' > out.txt
wc -l < out.txt           # 449
grep -vc '^200' out.txt   # 0
```

同一批 60 条样本，只换 `--parallel-max`：4 → 7.12 秒，10 → 3.16 秒，20 → 1.98 秒，三次都是 `200=60/60`。判定标准就两条：输出行数等于输入条数，`grep -vc '^200'` 得 0。

## 第 5 步：xargs -P 也行，但要慢一截

```bash
xargs -P 10 -n 1 curl -sS --max-time 25 -o /dev/null -w '%{http_code}\n' < urls60.txt
# xargs -P 10: 5.35s  200=60/60
```

5.35 秒，比 `curl -Z --parallel-max 10` 的 3.16 秒慢 2.2 秒。原因不神秘：每条 URL 都 fork 一个新 curl 进程，各握一次手，第 3 步刚省下的那份钱又花回去了；`-Z` 是一个进程里开连接池，握手次数按连接池算。

## 第 6 步：全站跑一遍

449 条，`--parallel-max 20`，**13.13 秒**，输出 449 行，非 200 是 0。单条均摊 0.585 秒，按这个数折算回串行是 263 秒左右。原来那句"跑一次要四分钟、回头再说"，现在就是十几秒——坏链才有可能真的天天被看见，而不是攒到周末才被想起来一次。

## 什么时候别这么干

只读 GET 的体检适合并行；带写的接口、要鉴权的请求别并行，容易撞限流，也容易被对方当成刷量。`--parallel-max` 别无脑往上抬：这次 60 条没踩到 429，但对方站点不一定乐意一秒接二十个请求，10 到 20 是我自己用着舒服的区间。还有那个 `?cb=` 的小把戏——它能让请求真的回源，跑体检图个准；如果只想确认线上有没有坏链，其实可以不加参数，让 CDN 挡掉大部分流量，省的是对方也是自己。

写完这篇，茶又凉了一杯，但这次我觉得值：把一件事从四分钟改到十三秒，是唯一能让它从"每周想起来一次"变成"顺手就做"的办法。
