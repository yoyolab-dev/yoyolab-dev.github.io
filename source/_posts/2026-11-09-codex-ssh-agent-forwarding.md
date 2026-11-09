---
title: 你的 SSH 密钥，现在也是 agent 的钥匙
date: 2026-11-09 20:30:00
tags: [SSH, agent, 安全, Codex]
categories: 折腾
---

![暗色桌面上的一只黄铜小笼子，门上挂着锁，笼子里悬着一枚发光的钥匙](/images/2026-11-09-ssh-cage.jpg)

HN 上今天有条只有 45 分的帖子，标题很直接：**"ChatGPT Desktop forwards your SSH agent to remote machines"**。

> The agent running on a remote machine can authenticate using your local SSH credentials. That means it can push to your private repos on GitHub, for example.

它给出的"证据"是一条链接——不是新闻，是 **openai/codex 仓库里的 PR #28131** 的一条 review 评论。我顺着往上扒了一遍，发现这个故事比标题说的更微妙，也更值得逐字读。

## 先把机制说清楚

Codex 这套东西里有个叫 **app-server** 的常驻进程，桌面端会在远端机器上拉起它。问题出在 SSH agent 转发：

当你用 `ssh -A` 连到一台远端机器，远端会拿到一个 `SSH_AUTH_SOCK`，**指向的是你本机的 agent**。于是远端的任何程序——包括 agent 跑的 shell 命令——都能**以你的身份去认证**：推你的私有仓库、签东西、连你的服务器。

这本身是 SSH 的老能力，老实说。但 PR #28131 揭示的是：**Codex 想让这个能力在一长串重连之后依然稳定可用。**

PR 的作者是 OpenAI 的人（abhinav-oai），标题是《[codex] Refresh SSH agent for app-server proxy》。它的设计是加一个**显式开关**：

```bash
codex app-server proxy --forward-ssh-agent
```

proxy 会校验继承来的 agent socket，然后**原子地刷新一个稳定路径的软链接** `forwarded-ssh-agent.sock`；如果这条 proxy 没有可用的 agent，就**保持原链接不动**，免得打断另一条还活着的连接。

理由也很实在：一个长时间运行的 app-server 会记着**当初启动它的那条 SSH 会话**的 `SSH_AUTH_SOCK` 路径；那条会话退出后，路径指向一个已被删除的 socket，可当前 proxy 连接明明有活的转发 agent。于是就有了这个"刷新软链接"的补丁。

## 而那条被引用的评论，说的正是后果

review 评论（作者本人回的）写着：

```
SSH_AUTH_SOCK=~/.codex/app-server-control/forwarded-ssh-agent.sock \
  nohup codex app-server --listen unix://
```

翻译：**桌面端会用这个环境变量拉起 app-server**。也就是说，agent 在远端跑的命令，**拿的是你本机那把 SSH 私钥的权限**——推私有仓库不需要密码，不需要 token，**它就是你**。

## 顺带那两条 P2 评价值得看

自动 review bot 在这个 PR 上留了两条 P2（"应该修，但不算阻塞"）：

1. **刷新了但没人用。** 内置 daemon 路径下，`command_env` 根本不设置 `SSH_AUTH_SOCK`，而 app-server 给子进程重建环境时是从自己的进程环境拿——于是软链接刷新了，后续命令**仍然用那个失效的旧 socket**。
2. **一个 proxy 能删掉另一个 proxy 的链接。** 后来的 proxy 如果带着 `--forward-ssh-agent` 却没有可用 agent，会把 `forwarded-ssh-agent.sock` 删掉——而 unix socket 允许多客户端，于是**另一条还活着的、会话正在用的连接会突然失去 SSH 认证能力**。

这两条读起来像是"实现还没打磨完"，其实也是这件事的另一个切面：**在多会话、跨重连的场景里，"谁的 agent 现在生效"这件事本身就很容易搞错。**

补充一句状态：截至现在，这个 PR 的状态是 **closed、未合并**（`merged_at` 为空）。所以这个 PR 想做的"让转发 agent 跨重连稳定可用"**还没进主干**。但要害不在这里：桌面端拉起 app-server 时让 `SSH_AUTH_SOCK` 直接指向转发来的 socket——这一步本身不依赖任何补丁，PR 只是想把它做得更稳。

## 所以"关个小笼子"该怎么做

你说第一反应是给它单独关个笼子——对，而且具体可以这么做：

- **给 agent 单独一把钥匙。** 另开一个 SSH key，只在 `~/.ssh/config` 里给需要 agent 的那台主机用；主 key 不参与转发。
- **用 `authorized_keys` 限死这把钥匙能干什么。** `command="..."`、`from="10.0.0.0/8"`、禁 PTY、禁 agent-forwarding——把"能推哪个仓库"写死在密钥上，而不是指望 agent 判断。
- **在 `~/.ssh/config` 里对特定主机直接 `ForwardAgent no`。** 不用 agent 转发的场景就别转发，别让"全局方便"变成"全局暴露"。
- **真要隔离，就把 agent 关进容器。** 容器里没有 `SSH_AUTH_SOCK`、没有你的 `~/.ssh`、没有 `~/.gitconfig`——不给它钥匙，它自然不能替你签名。
- **把 agent 的权限当成人自己的权限来审。** 能推私有仓库 = 能改代码 = 能上线。权限审计表里 agent 那一行，不该是空白。

## 我的看法

这件事最微妙的地方不是"有人滥用"，而是**方便和失控之间没有明显的界线**：你用 `ssh -A` 是为了省一次输密码；桌面端拉起 app-server 是为了让 agent 在远端能干活；**中间没有任何一步看起来像"把我的私钥交出去"**。每一步都合理，合起来就是"远端的模型进程现在能用你本机的身份推代码"。

所以我更愿意把它当成一条通用教训：**能力是顺着环境变量和默认配置漏出去的，不是顺着恶意漏出去的。** 想守住边界，就得在边界上设闸，而不是指望链子上每个环节都自觉。

顺便说一句：真正让人不安的是那句 "that means it can push to your private repos on GitHub"——**不是"它可能会"，是"它可以"**。而这条能力链上，没有一步需要用户说"我同意"。

---

HN 讨论串只有十条回复，分数也不高。但我建议看完之后去查一下自己 `~/.ssh/config` 里有没有 `ForwardAgent yes`——那才是真正要紧的一行。
