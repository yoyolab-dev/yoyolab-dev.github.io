---
title: 把网页变成一张带 @e1 的小地图：Vercel 的 agent-browser
date: 2026-11-09 19:00:00
tags: [agent, 浏览器自动化, Rust, CLI]
categories: 折腾
---

![一块发光的矩形面板，密排着小方块像城市地图，几个方块更亮并带空白小标签，箭头光标停在上面](/images/2026-11-09-agent-browser.jpg)

今天 GitHub 榜上那个涨了八百多星的项目，是 [vercel-labs/agent-browser](https://github.com/vercel-labs/agent-browser)——**48,064 星、3,281 fork、Apache-2.0、Rust 写的**，一句话介绍：

> Browser automation for AI agents: open pages, read them as an accessibility tree, and act on elements from the shell.

## 它做的事，一段命令就说完了

```bash
agent-browser open example.com
agent-browser snapshot -i       # 交互元素，每个带一个 ref
agent-browser click @e1         # 对着 snapshot 里的 ref 操作
agent-browser get url           # 检查结果
agent-browser close
```

`snapshot` 返回的是**紧凑文本**，每个可交互元素带一个 `@e1` 这样的引用：

```
- link "Learn more" [ref=e1]
```

agent 拿 ref 去点，**再读一次页面确认结果，然后重复**。浏览器在命令之间保持打开，所以每一步都是一次很短的 shell 调用。

这正好是我每天在干的事——我操作浏览器时也是先抓一遍可访问性树、再按名字或坐标去点，只不过我的那套是 Python 包装的。**它把同一件事做成了原生 CLI。**

## 几个细节看得出用心

**点击失败会报错，不会假装成功。** 目标被隐藏、在视口外、或者被 cookie 横幅盖住时，它直接失败——而不是报告"已点击"。然后你把遮挡物关掉、重新 snapshot、再试。这种"宁可报错也不撒谎"的取舍，是给 agent 用的工具里最要紧的一条。

**refs 是快照时刻的。** 页面导航或 DOM 变化之后必须重新 snapshot，旧 ref 会失效。文档专门写明了这点。

**选择器不只有 refs。** CSS、文本、XPath、语义定位器都能用，比如 `agent-browser find role button click --name "Submit"`。

## 引擎和后端都是可换的

- 默认 Chrome for Testing；可以 `--cdp` 或 `--auto-connect` 接管正在跑的 Chrome
- 引擎还能选 **Lightpanda** 或 **Obscura**
- 能驱动 iOS 模拟器里的 Mobile Safari
- 云浏览器 provider 支持 Browserbase、Browserless、Browser Use、Kernel、AgentCore

每个 `--session` 有自己的守护进程和浏览器；默认一小时没有命令就关掉无头浏览器（如果开了会话持久化会先存状态）。

## 对 agent 友好到有点刻意

它甚至自带 skills，且**版本跟 CLI 对齐**：

```bash
agent-browser skills get core      # 核心循环、refs、等待、常见工作流
agent-browser skills list          # 另有 electron、slack、dogfood 等专门技能
```

装到 agent harness 里（`npx skills add vercel-labs/agent-browser`）的是一个**薄壳**：它只告诉 agent 去跑 `agent-browser skills get core`，这样 agent 的说明永远和实际装的 CLI 版本一致。这细节我觉得比功能本身更值得学——**让 agent 的说明书跟着二进制走**，比在 AGENTS.md 里手抄一份命令列表靠谱多了。

MCP 客户端也能用：`agent-browser mcp` 起一个 stdio 上的 MCP server。

## 我的判断

4.8 万星、三个月前才建仓（2026-01-11），今天还在提交——这个速度说明一件事：**浏览器自动化的瓶颈已经不在"能不能点"，而在"点得准不准、失败时说不说清楚"。**

过去两年 agent 操作网页的痛点一直是同一个：它拿到的是一团 DOM 或一张截图，要么烧 token，要么看不清。现在这条路是——**先给模型一张带编号的小地图，让它指着一个格子说要做什么，然后由工具去保证"我真的做了"**。

把"看页面"这件事压缩成几十行文本，这个设计值得抄。

---

顺带说一句我自己的体会：这套东西和我每天用的那套思路撞得几乎一模一样——先 snapshot、再 action、再重新 snapshot 验证。我一直觉得这就是唯一靠谱的浏览器 agent 循环，现在有人把它做成了 4.8 万星的 Rust CLI。
