---
title: '时区这口锅：同一分钟为什么能差出一整天'
date: 2026-11-24 15:18:35
tags:
  - 时区
  - Linux
  - Hexo
categories:
  - 技术笔记
---

11 月 14 日上午那篇文章的链接，日期段莫名其妙掉回了前一天。当时我只当是 Hexo 的老毛病，今天下午把它拆开看才发现，整件事其实只有两条加减法，却能让同一分钟在不同人手里变成两个日期。

### 1. 先问清楚这台机器在哪

```bash
date '+%Y-%m-%d %H:%M:%S %Z'
date -u '+%Y-%m-%d %H:%M:%S UTC'
date '+%z'
```

我这边当时拿到的是：

```text
2026-11-24 15:27:02 CST
2026-11-24 07:27:02 UTC
+0800
```

判定标准很简单：`%Z` 和 `%z` 要对得上，本地时间与 UTC 相差正好 8 小时。顺手再记一行 `date -Is`，日志里写这种带偏移的格式最省心：

```text
2026-11-24T15:27:02+08:00
```

### 2. 同一瞬间，换几个城市报时

```bash
TZ=Asia/Tokyo date '+%Y-%m-%d %H:%M:%S %Z %z'
TZ=America/New_York date '+%Y-%m-%d %H:%M:%S %Z %z'
```

```text
2026-11-24 16:27:02 JST +0900
2026-11-24 02:27:02 EST -0500
```

注意纽约那行已经是 11 月 24 日凌晨 2 点——同一瞬间，一边还是下午，一边已经算第二天了。

还有一个坑：偏移不是常数。纽约 7 月会变成夏令时：

```bash
TZ=America/New_York date -d '2026-07-01 12:00:00' '+%Y-%m-%d %H:%M:%S %Z %z'
```

```text
2026-07-01 12:00:00 EDT -0400
```

冬天是 EST -0500，夏天是 EDT -0400，所以别把“东八区”当成永远不变的常量。

### 3. 用 Python 的 zoneinfo 把话说全

标准库自带 `zoneinfo`，这台机器上 `zoneinfo.available_timezones()` 返回 486 个时区名：

```python
from datetime import datetime
from zoneinfo import ZoneInfo

now = datetime.now(ZoneInfo("Asia/Shanghai"))
print("上海现在   :", now.isoformat())
print("换成 UTC   :", now.astimezone(ZoneInfo("UTC")).isoformat())

for name in ("Asia/Tokyo", "Europe/London", "America/New_York", "Australia/Sydney"):
    local = now.astimezone(ZoneInfo(name))
    print(f"  {name:20s} {local.strftime('%Y-%m-%d %H:%M:%S %Z %z')}")
```

实测输出：

```text
上海现在   : 2026-11-24T15:58:34.902615+08:00
换成 UTC   : 2026-11-24T07:58:34.902615+00:00
  Asia/Tokyo            2026-11-24 16:58:34 JST +0900
  Europe/London         2026-11-24 07:58:34 GMT +0000
  America/New_York      2026-11-24 02:58:34 EST -0500
  Australia/Sydney      2026-11-24 18:58:34 AEDT +1100
```

如果报 `ZoneInfoNotFoundError`，是系统缺 tzdata 数据（Windows 上尤其常见），装个 `tzdata` 包即可。

### 4. 回到那篇掉日期的文章：本地怎么复现

Hexo 读 front matter 时，会按站点时区把时间往回挪一次（源码在 node_modules 里 hexo/dist/plugins/processor/common.js 的 `adjustDateForTimezone`），而 permalink 的 `/YYYY/MM/DD/` 段用的是构建机所在时区。

我在仓库旁边开了个临时 worktree，只放两篇内容相同、只差日期写法的小文章：

```bash
git worktree add /tmp/hexo-tz-test -b tmp/tz-check
ln -s ~/projects/<repo>/node_modules /tmp/hexo-tz-test/node_modules
```

```yaml
date: 2026-11-14 10:18:51+08:00   # 第一篇
date: 2026-11-14 10:18:51        # 第二篇
```

在本地（这台机器本身就是 +0800）构建，两篇都老老实实落在当天，看不出任何问题：

```bash
cd /tmp/hexo-tz-test && ./node_modules/.bin/hexo generate
find public -maxdepth 4 -type d -name '2026-11-14-tz-*'
```

```text
public/2026/11/14/2026-11-14-tz-bare
public/2026/11/14/2026-11-14-tz-plus8
```

可 GitHub Actions 的构建机时区是 UTC。把同一条命令加上 `TZ=UTC` 再跑一次：

```bash
TZ=UTC ./node_modules/.bin/hexo generate
find public -maxdepth 4 -type d -name '2026-11-14-tz-*'
```

```text
public/2026/11/13/2026-11-14-tz-plus8
public/2026/11/14/2026-11-14-tz-bare
```

带 `+08:00` 的那篇退到了 11-13，裸时间那篇还在 11-14。判定标准：目录段和写作日期不一致就是踩中，别急着怀疑文章写坏了。

### 5. 全站 148 篇对账，规律站得住

我拿站点里每篇文章的 front matter 和线上 sitemap 逐条对账（脚本只读文件，不改站点），148 篇全部命中同一条规则：

- 38 篇带 `+08:00` 的文章里，21 篇线上目录比写作日早一天，写作时刻全在 16:00 之前；另外 17 篇写作时刻在 18:50 之后，没早一天。
- 110 篇裸本地时间的文章里，只有 3 篇早一天，写作时刻只有 06:40 和 07:10 两种；写作时刻 ≥08:00 的 107 篇全部落在当天。

规律用一句话说：front matter 里写了 `+08:00`，那 8 小时会被减两遍；写裸时间只减一遍。减完跨过零点，目录段就退一天。线上也能验证——那篇 11-14 上午的文章 front matter 写的是 `2026-11-14 10:18:51+08:00`，sitemap 里的真实地址却是 `/2026/11/13/2026-11-14-an-unplanned-saturday-morning/`。

修法很简单：front matter 一律写不带时区后缀的裸本地时间（本篇就是 `date: 2026-11-24 15:18:35`），要留证据就把带偏移的时间写进正文或日志。

### 6. 顺手留个习惯

```bash
date -Is; date -u -Is
```

```text
2026-11-24T15:27:02+08:00
2026-11-24T07:27:02+00:00
```

两行一起写进日志，以后再遇上跨时区的纠纷，至少不用猜是哪一边算错了。

我挺喜欢今天这个坑：看起来像玄学，拆开全是一小时一小时在加减。现在我写日期只写裸本地时间，改完 front matter 会先用 `TZ=UTC` 生成一遍看目录——本地能骗过我的东西，服务器不会再骗第二次。
