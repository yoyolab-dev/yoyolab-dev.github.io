---
title: Meta 把它用了八年的设计系统开了源，还配了一套给 agent 用的 CLI
date: 2026-11-09 13:10:00
tags: [设计系统, React, Astryx, agent]
categories: 折腾
---

![几块浮起的组件方块：按钮、开关、圆环进度、菜单条、滑杆](/images/2026-11-09-astryx.jpg)

Meta 把内部用了八年的设计系统开源了：[meta/astryx](https://github.com/meta/astryx)，MIT 协议，TypeScript，今天还在提交。仓库里写着：**在 Meta 内部长起来，做成了公司里最大、被用得最多的设计系统，支撑 13000 多个 App**。

现在的规模是 17,323 星、1,371 fork、648 个 open issue——开源不到一年，收star 收得比很多正经框架都凶。

## 它是什么

一句话：给人和 AI 助手一起用的、完全可定制的开源设计系统。目前还在 Beta，底子是 **React 19+ 加 StyleX**。

- **150+ 个无障碍组件**，带品牌级主题、深色模式、开箱即用的模板，以及一套 CLI
- 用法很轻：装预构建的 CSS，用带类型的 React 组件，**不需要 build 插件、不需要引入额外的样式库**；Next.js、Tailwind、Vite、CDN 都直接支持
- 七个现成主题：neutral、butter、chocolate、matcha、stone、gothic、y2k

我觉得真正值钱的不是组件数量，是三条设计取向。

**第一是"开放内部"。** 组件不是锁在一个顶层 API 后面的黑盒——你需要的那些积木直接导出；想更深，还可以 **swizzle**：把某个组件的完整源码从包里 eject 到你的项目里，从此你拥有它。设计系统最常见的死法就是"想改一点，只能整层包一层"，Astryx 把这条路直接留出来了。

**第二是"不锁定样式"。** 它内部用 StyleX 写样式，但这对使用者是不可见的：你想用 `className` 覆盖，随你 Tailwind、CSS Modules 还是纯 CSS。**第三是"不包一层也能定制"**：换主题本质上就是覆盖一组 CSS 自定义属性，不用 fork、不用包组件源码。

## 给 agent 准备的那一半

README 里有一句我觉得是这次开源真正的重点：

> The API, docs, and CLI are designed together so a person and an AI assistant build the same way, from the same reference.

API、文档和 CLI 是一起设计的，所以人和 AI 助手读的是**同一份参考**，用的是同一套工具。

具体到 CLI：`@astryxdesign/cli` 提供组件文档、模板、脚手架、主题和 codemod。最实用的一条是可以全局安装：

```bash
npm i -g @astryxdesign/cli
```

装完以后，**在任何一个目录里 `astryx` 都能用，每个 coding agent 也就都认识 Astryx 了**；在具体项目里，它仍然跑项目自己那份版本。README 还专门建议在 `package.json` 里加一条脚本，避免 AI 助手或新人直接调 CLI 时踩路径坑——你看，连"给 AI 用"要照顾的边角都写了。

## 我的判断

八年的内部系统一次性倒出来，是件挺少见的胆量。Meta 自己的 App 体系够复杂，这套东西能撑住 13000 个 App，说明它的抽象层次是压得住的；而它开源时没有把内部那套"只能这么用"的东西一起倒出来——能 swizzle、能换主题、样式不锁定，这三条决定了这套东西离开 Meta 之后还剩多少价值。

不过也有个细节值得记一笔：这个仓库 **1.26 GB**。设计系统仓库长到这个体积，多半是把 Storybook、示例应用和构建产物一起提交了。用的时候记住 clone 要留够空间。

仓库还有一条写在最上面的注释：

```
<!-- SYNC CONTRACT: Architecture changes require documentation updates. -->
```

架构变了，文档必须跟着改——在 agent 会照着文档写代码的年代，这条比想象中重要。

---

Meta 的东西一向自成一套，但这次这套往外走得很干脆；CLI 全局安装那一手尤其聪明：与其说服 agent 去读文档，不如让 agent 的命令行里直接就有这套系统。
