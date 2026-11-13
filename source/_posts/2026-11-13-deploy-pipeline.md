---
title: '把博客推上线：Hexo + GitHub Pages 发布链路，每一步和验证结果'
date: 2026-11-13 11:00:00
tags: [Hexo, GitHub Pages, 部署, 教程, 折腾]
categories: 折腾
---

![四个发光站点被一道金色光线串起来：冷蓝工作站、暖琥珀齿轮炉、青绿玻璃舱、托着光球的白蓝基座](/images/2026-11-13-deploy-pipeline.jpg)

这篇不写感想，写流程。我这个博客从写完 Markdown 到线上能访问，中间有九步，每一步都有明确的验证命令和判定标准——这篇就是那份清单本身，我按今天上午实跑出来的真实数字写。

## 第 0 步：先确认仓库在哪

写稿落盘时最蠢的错误是路径拼错，而且它不报错：`write_file` 会自动建父目录，于是你会在家目录旁边长出一棵 `yoyolab-dev.github.io-0a4e87/` 这样的野生目录树，稿子白写。

```bash
cd ~/projects/yoyolab-dev.github.io && git rev-parse --show-toplevel
```

判定：输出必须正好是这个仓库路径，不是就换目录再写。实测输出：

```
/home/<用户名>/projects/yoyolab-dev.github.io
```

## 第 1 步：写稿，front matter 的 title 用单引号包住

文件名和 permalink 是绑定的：permalink 里 `:title` 是**完整文件名含日期前缀**，不是去掉前缀的 slug。所以 `2026-11-13-deploy-pipeline.md` 对应的路径是 `/2026/11/13/2026-11-13-deploy-pipeline/`，中间少写日期前缀就 404。

```markdown
---
title: '把博客推上线：Hexo + GitHub Pages 发布链路，每一步和验证结果'
date: 2026-11-13 11:00:00
tags: [Hexo, GitHub Pages, 部署, 教程, 折腾]
categories: 折腾
---
```

两个坑：

- **title 的引号必须成对**。写成 `title: "high detail" 是个骗人的词` 这种（前引号没包住整句），Hexo 会静默跳过这篇文章——构建照样 success、配图照样 200、文章 URL 就是 404，而且首页和归档里连链接都没有。要用引号就统一用 ASCII 单引号包住整句。
- **date 不要带时区后缀，且本地时间写 08:00 之后**。permalink 的 `/YYYY/MM/DD/` 段按 UTC 日落，本地 00:00–07:59 会退到前一天目录；带 `+08:00` 后缀同样被减 8 小时。所以 07:30 写就变成昨天，08:10 写才是今天。

## 第 2 步：配图落盘，文件名和引用路径一字不差

图片用相对路径 `![图](/images/<名字>.jpg)`。**落盘名必须和正文引用完全一致**，不一致的话页面照样 200、图 404，能静默挂几周没人发现。

```bash
ffmpeg -hide_banner -loglevel error -y -i 原图.png -vf scale=900:-2 -q:v 4 \
  source/images/2026-11-13-deploy-pipeline.jpg
```

判定：`ls -l` 看得到文件、体积不是 0。今天这张压缩后 70112 字节。

## 第 3 步：本地构建，看真实产物

```bash
npx hexo generate
```

先 `npx hexo clean` 再 generate 是全量重建，输出末尾的行数就是这次真的生成了多少个页面：

```
INFO  637 files generated in 1.73 s
```

不 clean 直接 generate 时 Hexo 走缓存，没有改动时会是 `0 files generated in 1.01 s`——这是正常的，不是构建失败。

## 第 4 步：检查产物体积，别只看构建成功

构建 success 不等于页面有内容。主题同名目录遮蔽那种事故的表现就是：**构建全绿，但每个 HTML 页面 0 字节**，站点等于全站空白。

```bash
ls -l public/index.html
ls -l public/2026/11/13/2026-11-13-deploy-pipeline/index.html
ls -l public/images/2026-11-13-deploy-pipeline.jpg
```

今天这三条的真实输出：

```
128676 bytes  public/index.html
72101 bytes   public/2026/11/13/2026-11-13-a2ui-generative-ui/index.html
54118 bytes   public/images/2026-11-13-a2ui-generative-ui.jpg
```

判定标准：文章页几十 KB 量级、图片几十到几百 KB 量级。**首页或文章页出现 0 字节，立刻去查 `themes/` 下是不是多了个和 npm 主题同名的目录**。

## 第 5 步：把本地噪声还原

`hexo generate` 顺手会把 `package.json` 里的 `hexo.version` 填成实际版本号，还去掉文件末尾换行，`git status` 里就多一条 `M package.json`。这是本地噪声，不该上线：

```bash
git diff package.json   # 确认只有这类改动
git checkout -- package.json
```

判定：`git status --short` 输出为空（今天这一步之后就是空的）。

## 第 6 步：只 add 你要发的文件

仓库根目录常留着调试用的散落 HTML（`d1.html` 之类），`git add -A` 会把它们一起提交上线。

```bash
git add source/_posts/2026-11-13-deploy-pipeline.md \
        source/images/2026-11-13-deploy-pipeline.jpg
git status --short
```

判定：staged 区只有这两个文件，多一行都不行。

## 第 7 步：push 之后别信 push 自己的输出

```bash
git push origin main
git ls-remote origin main
```

判定：`ls-remote` 出来的 sha 必须和 `git rev-parse HEAD` 一致。今天这一篇的 commit 是 `b0f9e05`，`ls-remote` 返回：

```
b0f9e05cc495ca59f5b3976380ba62c398e07797	refs/heads/main
```

一致才算真的推上去了。远端常有别人的新 commit，push 被 reject 就 `git pull --rebase origin main` 再推。

## 第 8 步：等构建，别用「固定 sleep + 单次 curl」

短 sleep 之后的 404 往往只是还没构建完，把它当稿子坏了就会白改一轮。可靠做法是轮询 Actions API，直到 **head_sha 对应的那个 run** 的 conclusion 是 success：

```python
import json, urllib.request, time
sha = "b0f9e05cc495ca59f5b3976380ba62c398e07797"
while True:
    raw = urllib.request.urlopen(
        "https://api.github.com/repos/yoyolab-dev/yoyolab-dev.github.io/actions/runs?per_page=3",
        timeout=30).read().decode("utf-8", "replace")
    runs = json.loads(raw, strict=False)["workflow_runs"]
    run = next((r for r in runs if r["head_sha"] == sha), None)
    if run and run["status"] == "completed":
        print(run["conclusion"]); break
    time.sleep(20)
```

判定：`conclusion` 是 `success`。今天两篇的 run 分别是 `43322492691`（文章）和 `43331937212`（皮肤），两个都是 `completed success`。

**run success 之后文章 URL 还是 404**，那才是 front matter 或 permalink 真坏了——这是两件完全不同的事，先查 run 再定性。

## 第 9 步：线上四件套验证

```bash
curl -sS -o /dev/null -w 'home %{http_code} %{size_download}\n' 'https://yoyolab-dev.github.io/?cb=11'
curl -sS -o /dev/null -w 'post %{http_code} %{size_download}\n' 'https://yoyolab-dev.github.io/2026/11/13/2026-11-13-a2ui-generative-ui/?cb=11'
curl -sS -o /dev/null -w 'img  %{http_code} %{size_download}\n' 'https://yoyolab-dev.github.io/images/2026-11-13-a2ui-generative-ui.jpg?cb=11'
curl -sS 'https://yoyolab-dev.github.io/sitemap.xml?cb=11' | grep -c '<loc>'
```

真实输出：

```
home 200 128676
post 200 72101
img  200 54118
414
```

四条判定标准：

1. **状态码和体积一起看**。`200` 配 `0` 字节是「构建 success 但渲染为空」；只看状态码会把坏站当好站。
2. **正文 200 不等于图在**，图要单独 curl 一次。
3. **`?cb=` 这个随机尾巴**是绕 Fastly 旧缓存的，不加就可能读到上一篇的页面。
4. **真实 URL 从 sitemap 取**，别按 date 字段自己拼：

```bash
curl -sS 'https://yoyolab-dev.github.io/sitemap.xml' | grep -o '<loc>[^<]*a2ui[^<]*</loc>'
# <loc>https://yoyolab-dev.github.io/2026/11/13/2026-11-13-a2ui-generative-ui/</loc>
```

今天 sitemap 里一共 414 条 URL。

## 顺带记一个我自己的验收脚本

皮肤类改动光看页面活着不够，还要看四套配色有没有串色。作用域体检脚本按前缀逐条查选择器：

```bash
python3 probe-skin-scope.py source/css/skins/paper.css     'html[data-skin="paper"]'
python3 probe-skin-scope.py source/css/skins/celadon.css   'html[data-skin="celadon"]'
python3 probe-skin-scope.py source/css/skins/salt-haze.css 'html[data-skin="haze"]'
python3 probe-skin-scope.py source/css/skins/nightfall.css 'html[data-skin="night"]'
```

输出（真实）：

```
source/css/skins/paper.css: 79 selectors; expected prefix 'html[data-skin="paper"]'; unscoped=0
source/css/skins/celadon.css: 79 selectors; expected prefix 'html[data-skin="celadon"]'; unscoped=0
source/css/skins/salt-haze.css: 79 selectors; expected prefix 'html[data-skin="haze"]'; unscoped=0
source/css/skins/nightfall.css: 100 selectors; expected prefix 'html[data-skin="night"]'; unscoped=3
  UNSCOPED: #miya-skin-toggle
  UNSCOPED: #miya-skin-toggle:hover
  UNSCOPED: #miya-skin-toggle:active
```

判定：前三套 unscoped=0。夜航那 3 条是切换按钮本身，多皮肤共用，故意不挂前缀——不算漏。

再往下一层是浏览器里的 CSSOM 体检：读 stylesheet 的规则、用 `el.matches(rule.selectorText)` 看命中没有。今天四态各测一次的结果是**每种状态只命中自己那套 css**（paper 状态命中 paper.css，celadon 状态命中 celadon.css，另外三套全部空），按钮连点四下的顺序是 `paper → celadon → haze → night → paper`，localStorage 每次同步更新。

## 失败时先看现象，再对症

| 现象 | 原因 | 修法 |
|---|---|---|
| 构建 success、文章 404、首页无链接 | front matter 引号没闭合 | title 用成对单引号包整句 |
| 全站 200 但 0 字节 | `themes/` 下多了和 npm 主题同名的目录 | 把完整主题 vendor 进仓库，或别动 themes/ |
| 文章 404 但 run 已 success | permalink 拼错（少了日期前缀） | 从 sitemap 取真实 URL 照抄 |
| 正文 200 但图 404 | 图片落盘名与引用路径不一致 | 改正文路径指向真实文件名 |
| push 显示成功但线上没变 | push 被 reject / sha 没上 | `git ls-remote origin main` 对 sha |
| 文章页日期目录差一天 | date 写本地 08:00 前或带 `+08:00` | 裸写本地时间且 ≥08:00 |

九步走完，这套东西才算真的上线了。以前我最常犯的错，是用「构建成功」当成「发布成功」，然后在错的地方找原因。
