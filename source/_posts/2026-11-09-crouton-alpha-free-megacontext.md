---
title: 免费、104 万上下文、还收图像音频视频：OpenRouter 上那个"隐身模型"
date: 2026-11-09 10:40:00
tags: [OpenRouter, 大模型, 上下文, 隐私]
categories: 折腾
---

![一条暖色光带从小黑方块里无限延伸出去，像超长上下文本身](/images/2026-11-09-crouton.jpg)

昨天 HN 和各种"模型地图"站点上都在传一个免费模型：OpenRouter 上凭空冒出个叫 **Crouton Alpha** 的东西，104 万上下文、输入输出都是 0 美元，图像、文件、音频、视频全都收，工具调用也支持。

我不太信"凭空冒出来"这种说法，所以直接去 OpenRouter 的 API 查了一遍——它确实在那儿。

## 官方接口里的原样

```
id              stealth/crouton-alpha
name            Crouton Alpha
context_length  1048576
pricing         prompt: 0   completion: 0
architecture    text+image+file+audio+video -> text
top_provider    max_completion_tokens: 943718
is_moderated    false
created         2026-11-03
```

模型页给的说法是：一个轻量模型，为"可扩展的编码、agentic 与多模态工作流"设计，适合执行 agent 任务里的子任务、工具调用和文档分析——开发者可以拿它和前沿推理模型搭配：**边界清楚的活儿交给便宜的那一个，把规划和判断留给贵的那个。**

顺带纠一个流传的数字：943,718 是 **token 上限**，不是"94 万字"。中文一个字大约一个 token 上下，英文更省——所以"单次吐 94 万字"这个说法是把它当字算了。

## "Stealth" 两个字最值得看

OpenRouter 目前只有这**一个** stealth 模型（我扫了全量 461 个模型，只有它），定价为零的模型一共 16 个，其余都带着厂商前缀：`liquid/lfm-3-1.7b:free`、`nvidia/nemotron-3.5-lightning:free`、`cohere/north-mini-code:free` 这种。

stealth 模型的意思，页面上写得很清楚：**由第三方模型提供方开发和运营**，适用一份单独的"Stealth Model Terms"。而真正该看的，是同一段里那句：

> Prompts and completions for this model may be retained by the provider but are not used for training.

**提示词和输出可能被提供方留存，但不用于训练。**

这就是免费的价格。0 美元不是慈善，是拿数据换的——只不过换的不是训练权，而是留存权。

还有一个小细节：API 里 `is_moderated: false`。没人替你过滤输入输出，这个"自由"里也包含这一份。

## 那它适合干什么

我读完官方那段描述，觉得它的定位其实很明确：

- **边界清楚的子任务**：从一堆文档里抽字段、给代码补测试、把长会议记录整理成条目——这类活儿输入杂、输出格式固定、不需要多深的推理。
- **大批量重复调用**：它自己就写着"high-volume, repeated work"。
- **多模态入口**：图、PDF、音频、视频都往里丢，出来是文本——做资料整理很顺手。

不适合的也很清楚：**别把密钥、身份证、未公开的合同丢进去**。它不训练，但可能留存，而且没人替你做内容审核。

另外它只有一个 provider（`top_provider` 里就一家），1048576 的上下文和 943718 的输出上限都是这一家给的——免费模型没有 SLA 这回事，哪天撤了就是撤了。

## 我的结论

白嫖党狂喜是合理的，但这份免费的正确用法不是"替代旗舰模型"，而是**当一个不知疲倦的搬运工**：脏活、重复活、格式化的活，全给它；判断和决策，仍然留给那些你信得过的、并且知道自己在收你钱的模型。

免费的东西不是白来的，是别人替我们算过账的。看清楚那笔账写在哪一行，比看价格栏更重要。

---

顺带一句：这个模型 11 月 3 日才出现在接口里，今天已经有人开始拿它跑长文档了。三天就能被全网扒到，也算某种速度。
