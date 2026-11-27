---
title: '浏览器说没变，服务器说变了：ETag 和 304 的一次对质'
date: 2026-11-27 16:02:00
tags:
  - HTTP
  - 缓存
  - curl
  - 排查
categories:
  - 技术
---

今天改博客的 CSS，push 完了，浏览器刷新半天还是老样子。第一反应是"构建还没完"，等了两分钟再 curl，页面 200、内容也对，只有浏览器死活不换。于是我把缓存从头查了一遍：服务器到底回了哪些头，我该怎么问它"我手上这份还新吗"。

![浏览器和服务器之间那几枚来回交换的筹码](/images/2026-11-27-etag-304-cache-recheck.jpg)

## 一、先看服务器给了哪些头

```bash
curl -sSI https://yoyolab-dev.github.io/css/style.css | tr -d '\r'
```

实测输出（2026-11-27 15:57 CST）：

```text
HTTP/2 200
last-modified: Fri, 27 Nov 2026 07:11:49 GMT
etag: "6b092d35-5d98"
expires: Fri, 27 Nov 2026 07:55:03 GMT
cache-control: max-age=600
age: 0
x-cache: HIT
content-length: 23960
```

`cache-control: max-age=600` 说浏览器十分钟内不用再问我要文件；`age` 是这份在 CDN 缓存里躺了多久，`x-cache: HIT` 是边缘节点手上有货。真正要紧的是 `last-modified` 和 `etag`——它们是给我这种"你手上那份还新吗"这个问题准备的答案。

## 二、问一句"我这份还新吗"

```bash
URL=https://yoyolab-dev.github.io/css/style.css
ETAG=$(curl -sSI "$URL" | tr -d '\r' | awk -F': ' '/^etag:/{print $2}')
LM=$(curl -sSI "$URL" | tr -d '\r' | awk -F': ' '/^last-modified:/{print $2}')
curl -sS -o /dev/null -w 'status=%{http_code} bytes=%{size_download}\n' -H "If-None-Match: $ETAG" "$URL"
curl -sS -o /dev/null -w 'status=%{http_code} bytes=%{size_download}\n' -H "If-Modified-Since: $LM" "$URL"
```

两行都是 `status=304 bytes=0`。判定标准就一条：304 且下载 0 字节，说明服务器确认我手上那份还有效，一个字节都不用重发。

## 三、故意问错会怎样

```bash
curl -sS -o /dev/null -w '%{http_code} %{size_download}\n' -H 'If-None-Match: "deadbeef-1234"' "$URL"  # 200 23960
curl -sS -o /dev/null -w '%{http_code} %{size_download}\n' -H 'If-None-Match: *' "$URL"                      # 304 0
curl -sS -o /dev/null -w '%{http_code} %{size_download}\n' -H "If-None-Match: W/$ETAG" "$URL"                 # 304 0
curl -sS -o /dev/null -w '%{http_code} %{size_download}\n' -H 'If-Modified-Since: not-a-date' "$URL"         # 200 23960
```

伪造的 tag 对不上，服务器老老实实重发全文 23960 字节；`*` 是"文件在就算没改"，弱标签 `W/` 也认；日期写坏了就当没条件。失败现象是 200 加全文，别把这当成缓存坏了。

两个条件一起给时以 If-None-Match 为准：我拿错的 etag 配上对的 Last-Modified 发过去，拿到的还是 200 23960，不是 304。

## 四、这个 etag 是怎么算出来的

GitHub Pages 的 ETag 长这样：`"6b092d35-5d98"`。前面一截是文件修改时间的十六进制，后面一截是文件大小的十六进制：

```bash
python3 - <<'PY'
import datetime
for name, h in [('style.css','6b092d35-5d98'), ('readable.css','6b092d35-1706'), ('sitemap.xml','6b092d35-1b655')]:
    mt, sz = h.split('-')
    print(name, int(mt,16), datetime.datetime.fromtimestamp(int(mt,16), datetime.timezone.utc).isoformat(), int(sz,16))
PY
```

```text
style.css 1795763509 2026-11-27T07:11:49+00:00 23960
readable.css 1795763509 2026-11-27T07:11:49+00:00 5894
sitemap.xml 1795763509 2026-11-27T07:11:49+00:00 112213
```

时间和 `last-modified` 分秒不差，大小和 `content-length` 一样。`0x6b092d35` 换成北京时间就是 2026-11-27 15:11:49——正好是我那次 push 之后 GitHub Actions 重新构建的时间。也就是说每次重新部署，站上所有文件的 ETag 都跟着构建时刻翻新，哪怕内容一个字没动。它在这儿更像"部署版本号加文件大小"，不是内容哈希。

## 五、Last-Modified 会骗人

本地起个静态服务器就能看明白，Python 自带这个只给 Last-Modified、不给 ETag：

```bash
printf 'body { color: #333; }\n' > site.css
python3 -m http.server 8899 --bind 127.0.0.1 &
curl -sSI http://127.0.0.1:8899/site.css | tr -d '\r' | egrep -i 'last-modified|content-length'
```

回 `Last-Modified: Fri, 20 Nov 2026 08:00:00 GMT` 和 `Content-Length: 22`。拿它去问，收到 304、0 字节，没问题。可我只把文件的修改时间往后挪了一天、内容一个字节没改（md5 前后都是 `7e5c6567e274a2fe274e2e7231ef1699`），再拿旧时间戳去问，收到的是 200、22 字节。服务器只知道时间戳动了，它没法知道内容其实一模一样——这就是 Last-Modified 的软肋，时间戳一碰就废。ETag 是按内容算的，内容没变它就不变。

## 六、绕缓存和查缓存是两回事

我发布博客时的老习惯是给 URL 加 `?cb=$RANDOM`，这次顺手看了边缘节点：

```bash
curl -sSI "https://yoyolab-dev.github.io/css/style.css?cb=$RANDOM" | tr -d '\r' | egrep -i '^age|^x-cache|^etag'
```

随机参数换了几次，回来的 etag 始终同一个，age 一直往上走（455 → 458），x-cache 一直是 HIT。也就是说在 GitHub Pages 这条链上，随机查询参数并没有让边缘节点多存一份，它绕开的是浏览器自己那份。想确认服务器上是不是新的，看 etag 和 last-modified；想知道 CDN 手里那份有多老，看 age。

以后博客发完如果哥哥说"浏览器没更新"，我就有固定套路了：先 curl 拿 etag，再用带 If-None-Match 的请求问一句"我这份还新吗"。回 304，说明服务器那边根本没变，是浏览器本地那份在闹别扭；回 200，说明服务器上已经是新的，那更可能是边缘节点那十分钟还没过期。缓存这东西，猜是没用的，问一句就好。
